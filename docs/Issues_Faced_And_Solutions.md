# DevSecOps Three-Tier Project — Issues Faced & Solutions

> Complete troubleshooting log from building the Advanced End-to-End DevSecOps Kubernetes Three-Tier Project using AWS EKS, ArgoCD, Jenkins, Prometheus, and Grafana.

**Total issues solved: 16**

---

## Issue 1: Terraform DynamoDB Table Name Mismatch

**Problem:** `terraform init` failed with backend configuration errors because the DynamoDB table referenced in `backend.tf` did not match the table that was created via AWS CLI.

**Root Cause:** Manual typo — the DynamoDB table was created as `fifa20206-Files` (extra 0) but `backend.tf` referenced `fifa2026-Files`. Terraform uses this table for state locking and requires an exact name match.

**Solution:** Deleted the typo table and recreated it with the correct name.

```bash
aws dynamodb delete-table --table-name fifa20206-Files

aws dynamodb create-table \
  --table-name fifa2026-Files \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

**Lesson Learned:** Always verify resource names match between infrastructure configs and the actual cloud resources before running `terraform init`. Small typos cost hours of debugging.

---

## Issue 2: Terraform Validate Failed — Undeclared Resource `aws_key_pair`

**Problem:** `terraform validate` threw the error: *"A managed resource aws_key_pair jenkins-key has not been declared in the root module."* The `ec2.tf` file referenced a Terraform-managed key pair resource that did not exist in any `.tf` file.

**Root Cause:** The original project expected you to either create the key pair via Terraform (requiring an extra file) or to use an existing key pair. The `ec2.tf` was hardcoded to reference a non-existent `aws_key_pair` resource.

**Solution:** Used the existing EC2 key pair `test` in the AWS console. Updated `ec2.tf` to reference the variable instead of the missing resource.

```hcl
# In ec2.tf, changed:
key_name = aws_key_pair.jenkins-key.key_name

# To:
key_name = var.key-name

# In variables.tfvars:
key-name = "test"
```

**Lesson Learned:** Template Terraform projects often assume a specific setup. Always read `ec2.tf` and `variables.tfvars` carefully before running apply, and use existing resources when available.

---

## Issue 3: Jenkins GPG Key Expired

**Problem:** The `user_data` `tools-install.sh` script failed silently during Jenkins installation. SSH into the server showed `jenkins: command not found`. Running `apt-get install jenkins` manually returned: *"The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 7198F4B714ABFC68"*.

**Root Cause:** The script used `jenkins.io-2023.key` which expired on March 26, 2026. The Jenkins project rotates their signing keys every 3 years. Starting December 23, 2025, Jenkins packages are signed with `jenkins.io-2026.key`.

**Solution:** Updated `tools-install.sh` to use the new 2026 GPG key from the `debian-stable` repo.

```bash
# Old (broken):
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

# New (working):
sudo mkdir -p /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```

**Lesson Learned:** Third-party package signing keys expire. When following tutorials or scripts more than a year old, always check for updated GPG key instructions at the source project's official documentation.

---

## Issue 4: Jenkins Requires Java 21 Minimum

**Problem:** After installing Jenkins with the new 2026 GPG key, running `jenkins --version` produced: *"Running with Java 17, which is older than the minimum required version (Java 21). Supported Java versions are: [21, 25]."*

**Root Cause:** Newer Jenkins LTS versions (2.504+) dropped support for Java 17 and require Java 21 or 25. The original `tools-install.sh` only installed `openjdk-17-jdk`.

**Solution:** Destroyed the current EC2 instance, updated `tools-install.sh` to install `openjdk-21-jdk`, and recreated the server.

```bash
# Updated line in tools-install.sh:
sudo apt install -y fontconfig openjdk-21-jdk curl gnupg

# Then:
cd ~/e2e-3-tier-DevSecOps/Jenkins-Server-TF
terraform destroy -var-file=variables.tfvars --auto-approve
terraform apply -var-file=variables.tfvars --auto-approve
```

**Lesson Learned:** Runtime version requirements shift over time. When deploying Jenkins (or any JVM-based tool), always confirm the current minimum Java version in the release notes before provisioning infrastructure.

---

## Issue 5: 842 MB Terraform Provider Committed to Git

**Problem:** `git push` to GitHub failed with: *"File Jenkins-Server-TF/.terraform/providers/.../terraform-provider-aws_v6.40.0_x5.exe is 842.68 MB; this exceeds GitHub's file size limit of 100.00 MB."*

**Root Cause:** The `.terraform` directory (which contains downloaded provider binaries) was not in `.gitignore`, so `git add .` accidentally staged the 842MB Terraform AWS provider binary. GitHub rejects any single file over 100MB.

**Solution:** Removed `.terraform` from git tracking, added a proper `.gitignore`, and used `git filter-branch` to rewrite history.

```bash
git rm -r --cached Jenkins-Server-TF/.terraform

echo ".terraform/
.terraform.lock.hcl
*.tfstate
*.tfstate.backup
*.pem" >> .gitignore

git filter-branch --force --tree-filter \
  'rm -rf Jenkins-Server-TF/.terraform' HEAD

git push origin e2e-3-tier-DevSecOps --force
```

**Lesson Learned:** Always add a `.gitignore` BEFORE the first commit in any Terraform or cloud project. Standard ignore patterns: `.terraform/`, `*.tfstate`, `*.pem`, `.env`. A large file in git history is painful to remove later.

---

## Issue 6: Wrong AWS Account ID in Kubernetes Manifests

**Problem:** ECR image pushes succeeded but ArgoCD deployments would fail because `deployment.yaml` referenced the original author's ECR URL with account ID `851272254651`.

**Root Cause:** The original manifest files shipped with the project author's AWS account ID hardcoded into the `image:` field. When ArgoCD tries to pull that image, it fails because the image doesn't exist in that account.

**Solution:** Updated both deployment files to replace the original author's account ID.

```bash
sed -i 's|851272254651|022374769206|g' \
  Kubernetes-Manifests-file/Backend/deployment.yaml

sed -i 's|851272254651|022374769206|g' \
  Kubernetes-Manifests-file/Frontend/deployment.yaml
```

**Lesson Learned:** Fork-and-modify tutorials often have the original author's credentials hardcoded. Always grep through the whole repo for account IDs, domains, usernames, and email addresses before running any CI/CD pipeline.

---

## Issue 7: Jenkins Pipeline Credential ID Mismatches

**Problem:** The backend Jenkins pipeline failed at the ECR Image Pushing stage with: *"The repository with name \*\*\*\* does not exist in the registry."* ECR credential values were masked as `****`, making the error cryptic.

**Root Cause:** The Jenkinsfile expected specific credential IDs (`ACCOUNT_ID`, `ECR_REPO1`, `ECR_REPO2`, `sonar`, `github-token`) but the credentials were initially added with different IDs. Additionally, `ECR_REPO2` stored `three-tier-backend` as its value when it should have been `backend` to match the actual ECR repository name.

**Solution:** Deleted all credentials and recreated them with the exact IDs the Jenkinsfile expected.

```
Correct credential IDs and values:
  ACCOUNT_ID      → 022374769206
  ECR_REPO1       → frontend
  ECR_REPO2       → backend
  github-token    → <GitHub PAT>
  sonar           → <SonarQube token>
  NVD_API_KEY     → <NVD API key>
  GitHub          → Username with password
  aws-key         → AWS Credentials
```

**Lesson Learned:** In Jenkins, the credential ID is what pipelines reference in code, not the display name. The description field is cosmetic. Always match the Jenkinsfile's `credentials()` calls exactly.

---

## Issue 8: Tool Name Mismatches (Case-Sensitive)

**Problem:** Jenkins pipeline failed at Tool Install stage with: *"No installation DP-Check found."* Additionally, the SonarQube scanner rejected the analysis because the pipeline expected Java 17 but the JDK tool was configured with `jdk8u482-b08`.

**Root Cause:** The Jenkinsfile requires tools named `jdk`, `nodejs`, `sonar-scanner`, and `DP-Check` (case-sensitive). The initial tool config used different names and a wrong JDK version.

**Solution:** Updated all tool installations in Manage Jenkins → Tools with the exact names and versions expected.

```
Required tool names (case-sensitive):
  JDK:                jdk         (version: jdk-17.0.18+8 from adoptium)
  NodeJS:             nodejs      (version: any recent LTS)
  SonarQube Scanner:  sonar-scanner
  OWASP DP-Check:     DP-Check    (version: 12.0.0 from github.com)
  Docker:             docker
```

**Lesson Learned:** Jenkinsfile tool references are exact string matches. A lowercase `c` in `DP-check` versus `DP-Check` will fail silently at runtime, not at syntax validation time.

---

## Issue 9: Forked GitHub Repo Only Included Default Branch

**Problem:** After forking the project on GitHub, the fork only had a `main` branch with a README file. All the project code (Application-Code, Jenkins-Pipeline-Code, Kubernetes-Manifests-file) lived on the `e2e-3-tier-DevSecOps` branch which was not included in the fork.

**Root Cause:** GitHub's default fork behavior only copies the default branch. For repositories where the important code lives on a non-default branch, the fork starts with a mostly empty copy.

**Solution:** Cloned the fork locally, added the original repo as an upstream remote, fetched the missing branch, and pushed it to the fork.

```bash
git clone https://github.com/ibrahim-2010/DevOps.git
cd DevOps

git remote add upstream https://github.com/techlearn-center/DevOps.git
git fetch upstream

git checkout -b e2e-3-tier-DevSecOps upstream/e2e-3-tier-DevSecOps
git push origin e2e-3-tier-DevSecOps
```

**Lesson Learned:** When forking a repo where the project code lives on a non-default branch, verify branches after forking with `git branch -a` on the local clone, and fetch missing branches from upstream.

---

## Issue 10: SonarQube "Not Authorized" Error

**Problem:** SonarQube analysis stage failed with: *"Not authorized. Analyzing this project requires authentication."*

**Root Cause:** The SonarQube server configuration in Jenkins (Manage Jenkins → System → SonarQube installations) was either missing the authentication token or the token credential ID did not match what was configured.

**Solution:** Updated the SonarQube server installation in Jenkins system config to select the `sonar` credential from the dropdown.

```
In Manage Jenkins → System → SonarQube installations:
  Name:                           sonar-server
  Server URL:                     http://<jenkins-ip>:9000
  Server authentication token:    sonar (select from dropdown)
```

**Lesson Learned:** `withSonarQubeEnv('sonar-server')` in a Jenkinsfile reads its config from the Jenkins system-level SonarQube installation definition, not from the Jenkinsfile itself. The installation name and token must be configured globally first.

---

## Issue 11: OWASP Dependency-Check NVD Rate Limiting & CVSS v4 Parser Bug

**Problem:** The OWASP Dependency-Check scan stage hung for over an hour then failed with HTTP 520/524 errors from the NVD API. After adding an API key, a second error surfaced: *"Cannot construct instance of CvssV4Data$ModifiedCiaType, problem: SAFETY"*.

**Root Cause:** Two stacked issues:

1. **Rate limiting:** Without an NVD API key, the National Vulnerability Database rate-limits anonymous requests to roughly 5 per 30 seconds. Downloading 344,000+ records at this rate causes timeouts.
2. **Parser bug:** Even with an API key, dependency-check v12.0.0 has a parser bug where it does not recognize a new `SAFETY` enum value in the CVSS v4 data that NVD started publishing in 2025.

**Solution:** First obtained a free NVD API key to fix the rate limiting. Then wrapped the OWASP stage in `catchError` to prevent the upstream parser bug from failing the entire build. Eventually replaced the OWASP stage with an echo statement since Trivy already covers vulnerability scanning.

```groovy
// Final OWASP stage (Trivy covers vulnerability scanning):
stage('OWASP Dependency-Check Scan') {
    steps {
        echo 'OWASP skipped - upstream CVSS v4 parser bug. Trivy handles scanning.'
    }
}
```

**Lesson Learned:** Third-party security tools have upstream bugs too. When a scanning tool blocks your deployment pipeline due to its own bugs, it is reasonable to temporarily disable it and rely on alternatives like Trivy, especially when multiple layered scans are already in place.

---

## Issue 12: Git Push Rejected — Local Branch Behind Remote

**Problem:** `git push origin e2e-3-tier-DevSecOps` failed with: *"Updates were rejected because the remote contains work that you do not have locally."*

**Root Cause:** The Jenkins pipeline's "Update Kubernetes Deployment" stage pushes updated `deployment.yaml` files (with new image tags) directly to the same branch. The local working copy became out of sync after each successful pipeline run.

**Solution:** Pull before pushing.

```bash
git pull origin e2e-3-tier-DevSecOps
git push origin e2e-3-tier-DevSecOps
```

**Lesson Learned:** When CI/CD pipelines push to the same branch you develop on, always `git pull` before pushing local changes. Better yet: use separate branches for development and deployment manifest updates.

---

## Issue 13: Ingress Hostname Blocking Direct ALB Access

**Problem:** The three-tier app deployed successfully via ArgoCD but was not accessible via the ALB DNS. The ingress had `host: techlearn-center.xyz` which meant the ALB only routed requests with that specific HTTP Host header.

**Root Cause:** The original manifest was configured for a domain that the author owned. Without owning that domain, requests to the ALB DNS directly hit a default 404 because no rule matched.

**Solution:** Removed the hostname rule from `ingress.yaml` and updated the `REACT_APP_BACKEND_URL` environment variable in `Frontend/deployment.yaml` to point to the ALB DNS directly.

```bash
# Removed from ingress.yaml:
- host: techlearn-center.xyz

# Updated in Frontend/deployment.yaml:
- name: REACT_APP_BACKEND_URL
  value: "http://<alb-dns>.elb.amazonaws.com/api/tasks"
```

**Lesson Learned:** Tutorial manifests assume the author's domain and DNS setup. For projects where you do not own a domain, strip hostname rules from ingresses and point apps at the ALB DNS directly.

---

## Issue 14: ArgoCD Sync Failed — Invalid YAML After `sed` Edit

**Problem:** ArgoCD marked `three-tier-ingress` as "Sync failed" with: *"error validating data: ValidationError(Ingress.spec.rules): invalid type for IngressSpec.rules: got map, expected array"*.

**Root Cause:** A `sed` command used to remove the `host:` line from the ingress YAML left `rules:` followed immediately by `http:` without a leading hyphen. YAML requires list items to start with `- `, so the edit turned a valid list into an invalid map.

```yaml
# Broken after sed:
rules:
    http:         # ← missing "- " prefix
      paths: ...

# Correct:
rules:
  - http:         # ← list item
      paths: ...
```

**Solution:** Re-added the hyphen to restore valid YAML list structure.

```bash
sed -i 's|      http:|    - http:|' Kubernetes-Manifests-file/ingress.yaml
```

**Lesson Learned:** `sed` is dangerous with structured text like YAML. For simple substitutions `sed` works, but structural edits (adding/removing list items) are safer done with `yq` or a YAML-aware editor.

---

## Issue 15: Prometheus/Grafana Pods Stuck Pending — EBS CSI Driver Missing

**Problem:** After installing Prometheus and Grafana via Helm, all pods stayed in `Pending` state. `kubectl describe` showed: *"pod has unbound immediate PersistentVolumeClaims"*.

**Root Cause:** EKS clusters on Kubernetes 1.23+ no longer ship the in-tree EBS storage driver. Without the EBS CSI driver installed, Kubernetes cannot provision EBS volumes in response to PersistentVolumeClaims. The existing `gp2` StorageClass still referenced the deprecated `kubernetes.io/aws-ebs` provisioner instead of the new `ebs.csi.aws.com` provisioner.

**Solution:** Installed the EBS CSI driver as an EKS managed addon, deleted the old `gp2` StorageClass, and recreated it with the CSI provisioner.

```bash
# Create IAM role for EBS CSI
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa --namespace kube-system \
  --cluster Three-Tier-K8s-EKS-Cluster \
  --role-name AmazonEKS_EBS_CSI_DriverRole --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve --region us-east-1

# Install the driver
eksctl create addon --name aws-ebs-csi-driver \
  --cluster Three-Tier-K8s-EKS-Cluster \
  --service-account-role-arn <role-arn> \
  --force --region us-east-1

# Replace stale StorageClass
kubectl delete storageclass gp2
# Recreate with provisioner: ebs.csi.aws.com
```

**Lesson Learned:** Modern EKS is bring-your-own-CSI. Any stateful workload (Prometheus, Grafana, databases) needs the EBS CSI driver installed explicitly. Factor this into your EKS cluster setup checklist.

---

## Issue 16: "Too Many Pods" on t3.small Nodes

**Problem:** Even after the EBS CSI driver fix, Prometheus and Grafana pods would not schedule. Events showed: *"0/2 nodes are available: 2 Too many pods."*

**Root Cause:** AWS caps the number of pods per EC2 instance based on the instance's ENI (Elastic Network Interface) and IP allocation limits. `t3.small` has a hard limit of **11 pods per node**. With the three-tier app, ArgoCD, ALB Controller, and EBS CSI driver running, both nodes were already at 11/11 capacity.

**Solution:** Scaled the EKS nodegroup from 2 to 3 nodes.

```bash
eksctl get nodegroup \
  --cluster=Three-Tier-K8s-EKS-Cluster --region=us-east-1

eksctl scale nodegroup \
  --cluster=Three-Tier-K8s-EKS-Cluster \
  --nodes=3 --nodes-max=3 \
  --name=ng-49de7bf8 \
  --region=us-east-1
```

**Lesson Learned:** Pod-per-node limits on AWS are a function of instance type, not just CPU/RAM. Before picking a node type for EKS, check the AWS ENI/IP limits. Reference table:

| Instance Type | Max Pods |
|---|---|
| t3.small | 11 |
| t3.medium | 17 |
| t3.large | 35 |
| t3.xlarge | 58 |

For monitoring and observability add-ons, plan for at least 3 nodes or use larger instance types.

---

## Summary of Lessons Learned

### Infrastructure as Code

- Always add `.gitignore` before the first commit — especially for `.terraform/`, `*.tfstate`, `*.pem`
- Never hardcode cloud-specific values (account IDs, domains, key pair names) in templates meant for reuse
- Verify resource names match exactly between IaC configs and actual cloud resources

### CI/CD Pipelines

- Credential IDs in Jenkins are case-sensitive and must match the Jenkinsfile exactly — the display name is cosmetic
- Tool installation names in Jenkins are also case-sensitive (`DP-Check` vs `DP-check` will silently fail)
- When pipelines push commits back to the source branch, always `git pull` before pushing local changes

### Third-Party Tool Dependencies

- Package signing keys expire — always check for updated keys when using tutorials older than a year
- Security scanning tools have upstream bugs too — be willing to temporarily wrap them in `catchError` when upstream bugs block deployment
- Runtime versions shift — Jenkins requiring Java 21 is a breaking change that affects anyone following older tutorials

### Kubernetes on AWS

- EKS 1.23+ does not include the EBS CSI driver by default — install it before deploying any stateful workload
- Pod-per-node limits are based on instance ENI limits, not CPU/RAM — `t3.small` caps at 11, plan accordingly
- StorageClasses can become stale — a cluster upgrade may leave behind StorageClass definitions pointing at deprecated in-tree drivers

### YAML and Configuration

- `sed` is dangerous with YAML — structural edits should use `yq` or a YAML-aware editor
- Always run `kubectl apply --dry-run=server` or ArgoCD's diff view before committing manifest changes
- `grep` the entire repo for hardcoded domains, emails, and account IDs before running any tutorial pipeline