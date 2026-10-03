# Book-My-Show Project — Errors, Troubleshooting & Fixes

> **Project:** Book-My-Show React/Node.js application with Jenkins CI/CD, SonarQube, OWASP Dependency-Check, Trivy, Docker/Docker Hub, AWS EKS/Kubernetes, and email notifications.
>
> **Repository:** https://github.com/Lasvanthi1/Book-My-Show
>
> **Source of this document:** current repository inspected with Firecrawl/GitHub plus the troubleshooting history from the project work. Items are separated into confirmed errors, warnings/pitfalls, and configuration issues so that warnings are not confused with failures.

---

## 1. Jenkins Server Disk Full — Root Filesystem at ~99%

### Symptom

The Jenkins server root filesystem became almost full, with only a very small amount of free space remaining.

This can break or destabilize:

- Jenkins builds
- workspace creation
- npm installation
- Docker builds
- Dependency-Check reports
- log generation

### Diagnosis

```bash
df -h
sudo du -sh /var/lib/jenkins/* | sort -h
```

### Fix

The EC2/EBS disk was expanded and the partition was extended using:

```bash
sudo growpart /dev/nvme0n1 1
```

Then grow the filesystem according to its type.

For ext4:

```bash
sudo resize2fs /dev/nvme0n1p1
```

For XFS:

```bash
sudo xfs_growfs -d /
```

Verify:

```bash
df -h
```

### Lesson

Increasing the EC2 volume size alone is not enough. The partition and filesystem also have to be expanded.

---

## 2. Jenkins Tools Configuration Confusion

### Problem

The pipeline uses Jenkins-managed tools such as:

```groovy
tools {
    jdk 'jdk17'
    nodejs 'node23'
}
```

and:

```groovy
environment {
    SCANNER_HOME = tool 'sonar-scanner'
}
```

The build depends on Jenkins knowing which installation each name refers to.

### Cause

The names in the Jenkinsfile are identifiers, not executable names that Jenkins automatically discovers.

### Fix

Configure matching installations under:

**Manage Jenkins → Tools**

Examples:

```text
JDK installation       → jdk17
NodeJS installation    → node23
SonarQube Scanner      → sonar-scanner
Dependency-Check       → DP-Check
```

### Lesson

`tools {}` provisions tools. Jenkins system configuration/credentials configure integrations and authentication. They solve different problems.

---

## 3. SonarQube Quality Gate — `PENDING`

### Symptom

The SonarQube analysis completed, but Jenkins remained waiting at:

```groovy
waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
```

### Cause

`waitForQualityGate` relies on Jenkins receiving the SonarQube webhook callback. If the webhook is missing, incorrect, unreachable, or the SonarQube/Jenkins configuration does not match, the quality-gate result can remain pending.

### Fix

Configure the SonarQube webhook to the Jenkins endpoint:

```text
<JENKINS_URL>/sonarqube-webhook/
```

Also verify that the Jenkins SonarQube server configuration name matches:

```groovy
withSonarQubeEnv('sonar-server')
```

and that the token/credential used by the pipeline exists.

### Verification

Check SonarQube:

**Administration → Configuration → Webhooks**

Check Jenkins:

**Manage Jenkins → System → SonarQube servers**

### Lesson

SonarQube analysis and the Jenkins quality-gate callback are two separate parts of the integration.

---

## 4. npm Deprecation Warnings

### Symptom

During `npm install`, messages appeared such as:

```text
npm warn deprecated ...
```

### Cause

Some transitive dependencies used by the older React/application dependency tree are deprecated.

### Important distinction

A deprecation warning is **not automatically a build failure** and does not mean the application has an exploitable vulnerability.

### Fix

Do not treat every warning as a pipeline failure. For long-term maintenance:

```bash
npm outdated
npm audit
```

Review upgrade compatibility before changing major versions.

---

## 5. npm Alias Warnings

### Symptom

Warnings appeared for packages/aliases such as:

```text
@jest/react-is-18
@jest/react-is-19
string-width-cjs
strip-ansi-cjs
wrap-ansi-cjs
```

### Cause

npm's audit/dependency processing can encounter package aliases and compatibility packages in older dependency trees.

### Fix

These messages were treated as warnings rather than as proof of a vulnerability. Review the final `npm audit` output separately if vulnerability status is required.

### Lesson

Do not confuse an npm alias/deprecation warning with an actual CVE finding.

---

## 6. `Unable to find node module` Warnings During Dependency Analysis

### Symptom

Dependency analysis produced messages about missing nested Node modules while scanning `node_modules`.

### Cause

The pipeline scans the whole workspace:

```bash
--scan ./
```

The workspace contains a large generated `node_modules` tree. Dependency/security analyzers can encounter nested modules, generated files, aliases, optional dependencies, and package layouts that are not straightforward to resolve.

### Fix / Improvement

Prefer a clean CI dependency installation and carefully define the scan scope/exclusions rather than indiscriminately scanning generated dependency directories.

Also inspect the actual Dependency-Check report for CVE findings instead of interpreting every analyzer warning as a vulnerability.

---

## 7. OWASP Dependency-Check — `.NET Assembly Analyzer` Error

### Error

```text
.NET Assembly Analyzer could not be initialized
The 'dotnet' executable could not be found
The dotnet 8.0 core runtime or SDK is required
```

### Cause

Dependency-Check attempted to initialize its .NET assembly analyzer even though this is a Node/React application.

### Fix for this project

Disable the assembly analyzer:

```groovy
dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit --disableAssembly',
                  odcInstallation: 'DP-Check'
```

### Alternative

Install the required .NET runtime/SDK if the project actually contains .NET assemblies that must be analyzed.

### Lesson

Disable analyzers that are irrelevant to the technology stack when they create unnecessary environment requirements.

---

## 8. OWASP Dependency-Check — Long RetireJS Scan

### Log

```text
[INFO] Finished RetireJS Analyzer (608 seconds)
```

### Meaning

608 seconds is approximately 10 minutes and 8 seconds.

RetireJS analyzes JavaScript libraries for known vulnerable versions.

### Important

This line is **not an error**. It means the RetireJS analyzer finished.

### Why it took so long

The pipeline scans the workspace and the project contains a large JavaScript dependency tree under `node_modules`.

### Improvement

Reduce unnecessary scan scope and avoid scanning generated content when it is not needed, while retaining appropriate package/dependency analysis.

---

## 9. OWASP Dependency-Check — Sonatype OSS Index Analyzer Warnings

### Symptom

Warnings appeared for files such as:

```text
bookmyshow-app/node_modules/@adobe/css-tools/dist/umd/adobe-css-tools.js
bookmyshow-app/node_modules/@ant-design/colors/dist/index.esm.js
bookmyshow-app/node_modules/@ant-design/colors/dist/index.js
bookmyshow-app/node_modules/@ant-design/icons-svg/es/asn/AccountBookFilled.js
```

with:

```text
[WARN] An error occurred while analyzing ... (Sonatype OSS Index Analyzer)
```

### Meaning

These messages indicate that the analyzer had trouble processing those files. They are **not themselves proof that those packages are vulnerable**.

### Fix / Investigation

1. Check the generated Dependency-Check report for actual CVE/CVSS findings.
2. Keep Dependency-Check current.
3. Reduce unnecessary filesystem scan scope where appropriate.
4. Avoid treating an analyzer warning as a vulnerability finding.

---

## 10. Trivy Filesystem Scan

### Pipeline step

```groovy
stage('Trivy FS Scan') {
    steps {
        sh 'trivy fs . > trivyfs.txt'
    }
}
```

### Potential issue

Because the scan starts at `.` after npm dependencies have been installed, the scan can inspect a large workspace including generated dependency files.

### Improvement

Keep the scan intentional and review `trivyfs.txt` for actual findings. Consider the desired scan scope rather than assuming every filesystem warning is a build failure.

---

## 11. Docker Socket Permission Error

### Symptom

Jenkins was unable to communicate with Docker because the Jenkins service user did not have permission to access:

```text
/var/run/docker.sock
```

### Cause

Jenkins runs as the `jenkins` user, while the Docker socket is normally restricted to root and the `docker` group.

### Preferred fix

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

Then verify:

```bash
sudo -u jenkins docker ps
```

### Temporary workaround encountered

The project documentation contains:

```bash
sudo chmod 666 /var/run/docker.sock
```

This can make Docker accessible but weakens the socket's access control and should not be the preferred production solution.

### Lesson

Fix Jenkins' group membership instead of making the Docker socket world-writable.

---

## 12. Docker Hub Credential Error — `Could not find credentials matching docker`

### Error

```text
ERROR: Could not find credentials matching docker
Finished: FAILURE
```

### Cause

The Jenkinsfile was requesting the credential ID:

```groovy
credentialsId: 'docker'
```

but Jenkins did not have a credential with that exact ID.

### Fix

Either create/use a Jenkins credential whose ID is exactly:

```text
docker
```

or change the Jenkinsfile to the actual credential ID, for example:

```groovy
withDockerRegistry(
    credentialsId: 'dockerhub_creds',
    ...
)
```

### Important distinction

These are different fields:

```text
Credential ID  ≠  Docker Hub username
```

Changing the Docker Hub username/password does not fix a Jenkinsfile that is requesting a nonexistent credential ID.

### Current project troubleshooting

The pipeline was subsequently changed to use:

```groovy
credentialsId: 'dockerhub_creds'
```

Therefore Jenkins must contain a credential whose **ID** is exactly `dockerhub_creds`.

---

## 13. Docker Registry URL Configuration

### Problem

The pipeline was configured with a Docker Hub repository browsing URL such as:

```text
https://hub.docker.com/repositories/lasvanthi
```

### Better registry endpoint

For `withDockerRegistry`, use the Docker registry endpoint appropriate for Docker Hub, for example:

```groovy
withDockerRegistry(
    credentialsId: 'dockerhub_creds',
    url: 'https://index.docker.io/v1/'
) {
    ...
}
```

### Lesson

A Docker Hub browser/repository URL and a Docker registry endpoint are not the same thing.

---

## 14. Docker Image Name Mismatch — `lasvanthi/bms` vs `kastrov/bms`

### Problem

Different parts of the project used different Docker image names.

The current Jenkinsfile2 builds/pushes:

```text
lasvanthi/bms:latest
```

but the current `deployment.yml` contains:

```yaml
image: kastrov/bms:latest
```

### Effect

Jenkins can successfully build and push one image while Kubernetes tries to pull another image.

This can result in:

```text
ImagePullBackOff
ErrImagePull
```

or deployment of an unintended image.

### Fix

Use the same image reference throughout the pipeline and Kubernetes manifest.

For example:

```yaml
image: lasvanthi/bms:latest
```

if that is the image being published by Jenkins.

### Lesson

The image name/tag is part of the CI/CD contract between Docker publishing and Kubernetes deployment.

---

## 15. Docker Build Dependency Handling

### Current Dockerfile approach

The Dockerfile copies the package manifests and installs specific PostCSS versions before the general dependency installation:

```dockerfile
COPY package.json package-lock.json ./
RUN npm install postcss@8.4.21 postcss-safe-parser@6.0.0 --legacy-peer-deps
RUN npm install
```

### Why this matters

The project has explicitly pinned:

```text
postcss 8.4.21
postcss-safe-parser 6.0.0
```

This was used to address dependency compatibility in the older React build stack.

### Improvement

For reproducible container builds, prefer a valid lock file and `npm ci` where the dependency tree supports it, while preserving any required compatibility constraints.

---

## 16. AWS Credentials Work in One User but Not Jenkins

### Symptom

AWS commands could work when executed manually as `ubuntu` but fail when the pipeline runs as the Jenkins service user.

Typical affected commands:

```bash
aws sts get-caller-identity
aws eks update-kubeconfig ...
```

### Cause

Jenkins runs as the `jenkins` user. AWS credentials configured for another Linux user are not automatically available to Jenkins.

### Fix

Configure credentials for the Jenkins execution context and verify as Jenkins:

```bash
sudo -su jenkins
aws sts get-caller-identity
```

Then configure the EKS kubeconfig:

```bash
aws eks update-kubeconfig \
  --name lasvanthi-eks \
  --region us-east-1
```

### Lesson

Always troubleshoot cloud credentials using the same Linux user that actually executes the Jenkins pipeline.

---

## 17. EKS `update-kubeconfig` / Kubernetes Access

### Pipeline commands

```bash
aws sts get-caller-identity
aws eks update-kubeconfig --name $EKS_CLUSTER_NAME --region $AWS_REGION
kubectl get nodes
```

### Common failure causes encountered during the project

- AWS credentials unavailable to Jenkins
- wrong AWS region
- wrong EKS cluster name
- insufficient IAM permissions
- kubeconfig created for a different user
- Jenkins environment not using the expected AWS configuration

### Fix / Verification

Run as Jenkins:

```bash
aws sts get-caller-identity
aws eks list-clusters --region us-east-1
aws eks update-kubeconfig --name lasvanthi-eks --region us-east-1
kubectl get nodes
```

---

## 18. EKS/IAM Permission Problems

### Symptom

EKS operations failed when the IAM principal did not have sufficient permissions.

### Cause

Creating/updating EKS involves multiple AWS APIs and resources, including EKS, EC2, IAM, CloudFormation and networking-related operations.

### Fix / Investigation

Check the exact failing AWS command and identify the denied action from the AWS error message. Do not assume that having AWS credentials means the user has all required permissions.

For a production implementation, prefer least-privilege IAM rather than broad administrator-style policies.

---

## 19. Gmail SMTP — OAuth vs App Password vs TLS/SSL Confusion

### Problem

Email configuration involved confusion about whether OAuth, an app password, SSL, and TLS all needed to be enabled together.

### Correct separation

**Authentication:**

- username/password or app password
- OAuth 2.0

**Transport security:**

- SSL/TLS
- STARTTLS

An app password and OAuth are alternative authentication mechanisms; they are not normally two credentials that must both authenticate the same SMTP session.

### Typical Gmail SMTP configurations

SSL/TLS:

```text
SMTP host: smtp.gmail.com
Port: 465
Security: SSL/TLS
Authentication: username + app password
```

STARTTLS:

```text
SMTP host: smtp.gmail.com
Port: 587
Security: STARTTLS
Authentication: username + app password
```

The exact Jenkins plugin fields must match the selected transport mode.

---


## 20. Jenkins Docker Credential Changed but Restart-from-Stage Still Failed

### Situation

The Docker credential was changed and the build was restarted from the failed stage, but Jenkins still reported:

```text
Could not find credentials matching docker
```

### Cause

The Jenkinsfile still requested:

```groovy
credentialsId: 'docker'
```

while the newly configured credential had a different ID.

### Fix

Make the Jenkinsfile and Jenkins credential ID identical.

For the current troubleshooting configuration:

```groovy
credentialsId: 'dockerhub_creds'
```

and Jenkins must contain:

```text
ID: dockerhub_creds
```

### Restart-from-stage guidance

If the only failure was credential lookup, restarting from:

```text
Docker Build & Push
```

is appropriate after the credential ID has been corrected.

---


## 21. Jenkins Restart from Stage — When It Is Appropriate

### Use it when

A failed stage depends on configuration that has been corrected and earlier stages do not need to be repeated.

Example:

```text
Clean Workspace
      ↓
Checkout
      ↓
SonarQube
      ↓
Quality Gate
      ↓
Install Dependencies
      ↓
Trivy
      ↓
Docker Build & Push  ← fixed credential here
      ↓
Deploy to EKS
```

After fixing only the Docker credential, restarting from **Docker Build & Push** avoids unnecessarily repeating earlier work.

### Be careful

If the earlier stages generate files required by the restarted stage, verify that Jenkins reconstructs/restores the required workspace state for the restart. If not, run a normal build from the beginning.

---

# Current Repository Configuration — Important Checks

The current `main` branch contains:

```text
bookmyshow-app/
BMS-Document.txt
Jenkinsfile1
Jenkinsfile2
README.md
deployment.yml
service.yml
```

The repository's Jenkinsfiles use:

```groovy
jdk 'jdk17'
nodejs 'node23'
```

The Kubernetes pipeline uses:

```text
EKS cluster: lasvanthi-eks
AWS region: us-east-1
Docker image: lasvanthi/bms:latest
```

But `deployment.yml` currently contains:

```yaml
image: kastrov/bms:latest
```

That image-reference mismatch should be corrected before relying on the EKS deployment stage.

---

# Recommended Troubleshooting Order

When a build fails, troubleshoot from the first failing stage rather than changing several unrelated configurations at once.

```text
1. Jenkins executor
        ↓
2. Workspace / disk space
        ↓
3. Git checkout / branch
        ↓
4. Jenkins tools
        ↓
5. SonarQube connection
        ↓
6. Quality Gate / webhook
        ↓
7. npm installation
        ↓
8. Dependency-Check
        ↓
9. Trivy
        ↓
10. Docker daemon / socket
        ↓
11. Docker credentials
        ↓
12. Docker build
        ↓
13. Docker push
        ↓
14. AWS credentials
        ↓
15. EKS kubeconfig
        ↓
16. kubectl / Kubernetes manifests
        ↓
17. Pod/image/service verification
        ↓
18. Email notification
```

Useful commands:

```bash
# Jenkins / host
systemctl status jenkins
df -h

# Git
git branch -a
git ls-remote https://github.com/Lasvanthi1/Book-My-Show.git

# Node / npm
node --version
npm --version
npm ci

# Docker
systemctl status docker
docker ps
sudo -u jenkins docker ps

docker images

# Docker Hub authentication
cat ~/.docker/config.json

# AWS
aws sts get-caller-identity
aws eks list-clusters --region us-east-1
aws eks update-kubeconfig --name lasvanthi-eks --region us-east-1

# Kubernetes
kubectl get nodes
kubectl get pods
kubectl get svc
kubectl get endpoints
kubectl describe pod <pod-name>
kubectl get events --sort-by=.lastTimestamp
```

---

# Error vs Warning Quick Reference

| Log / Symptom | Error? | Meaning |
|---|---:|---|
| `couldn't find remote ref master` | Yes | Wrong Git branch configured |
| `Waiting for next available executor` | Yes/blocking | Jenkins cannot allocate an executor |
| Root filesystem 99% full | Yes/risk | Jenkins host lacks disk space |
| SonarQube Quality Gate `PENDING` | Blocking | Webhook/integration problem |
| `npm warn deprecated` | No | Deprecated dependency warning |
| npm alias warnings | Usually no | Dependency/audit alias handling |
| `fsevents` optional warning | Usually no | Optional macOS dependency on Linux |
| `.NET Assembly Analyzer` missing dotnet | Yes for that analyzer | Disable irrelevant analyzer or install .NET |
| `Finished RetireJS Analyzer (608 seconds)` | No | Analyzer completed after a long scan |
| OSS Index analyzer warnings | Analyzer warning | Does not by itself prove a vulnerability |
| Docker socket permission denied | Yes | Jenkins user lacks Docker access |
| `Could not find credentials matching docker` | Yes | Jenkins credential ID mismatch/missing credential |
| `221 2.0.0 closing connection ... gsmtp` | No | Normal SMTP connection close |
| `ImagePullBackOff` | Yes | Kubernetes cannot pull the requested image |
| AWS `AccessDenied` | Yes | IAM principal lacks required permission |
| `kubectl` connection/auth failure | Yes | Kubernetes access/kubeconfig problem |
| SSH/SCP failure | Yes | Remote host/key/user/network problem |

---

# Final CI/CD Flow

```text
Developer
   |
   v
GitHub (main)
   |
   v
Jenkins
   |
   +--> Clean workspace
   |
   +--> Checkout main
   |
   +--> SonarQube analysis
   |
   +--> Quality Gate
   |
   +--> npm install / npm ci
   |
   +--> OWASP Dependency-Check
   |
   +--> Trivy filesystem scan
   |
   +--> Docker build
   |
   +--> Docker Hub push
   |
   +--> AWS authentication
   |
   +--> EKS kubeconfig
   |
   +--> kubectl apply
   |
   v
Kubernetes Deployment
   |
   +--> Pods
   |
   +--> Service / LoadBalancer
   |
   v
Book-My-Show application
   |
   v
Jenkins email notification
```

---

# Key Lessons From the Project

1. **Credential ID matters:** Jenkins looks up the credential by ID, not by username.
2. **Git branch names must match exactly:** `main` and `master` are different refs.
3. **Jenkins tools are named installations:** the Jenkinsfile references the configured installation names.
4. **Authentication and encryption are separate:** app passwords/OAuth authenticate; TLS/SSL secures SMTP transport.
5. **Warnings are not automatically failures:** npm deprecations, optional dependencies, analyzer warnings, and normal SMTP responses must be interpreted correctly.
6. **Security scanners can be noisy:** inspect the generated vulnerability reports instead of treating every analyzer warning as a CVE.
7. **Jenkins runs as a service user:** Docker and AWS permissions must work for `jenkins`, not only for `ubuntu`.
8. **Docker image references must be consistent:** the image pushed by Jenkins must be the image Kubernetes deploys.
9. **Disk capacity is part of CI reliability:** large npm/Docker/security scans can consume Jenkins storage quickly.
10. **Restart-from-stage is useful after isolated configuration fixes:** for example, a corrected Docker credential can justify restarting at `Docker Build & Push`.

---

## Repository Evidence

- Repository: https://github.com/Lasvanthi1/Book-My-Show
- Jenkinsfile1: https://github.com/Lasvanthi1/Book-My-Show/blob/main/Jenkinsfile1
- Jenkinsfile2: https://github.com/Lasvanthi1/Book-My-Show/blob/main/Jenkinsfile2
- Kubernetes Deployment: https://github.com/Lasvanthi1/Book-My-Show/blob/main/deployment.yml
- Kubernetes Service: https://github.com/Lasvanthi1/Book-My-Show/blob/main/service.yml
- Application package.json: https://github.com/Lasvanthi1/Book-My-Show/blob/main/bookmyshow-app/package.json
- Dockerfile: https://github.com/Lasvanthi1/Book-My-Show/blob/main/bookmyshow-app/Dockerfile

> **Security note:** The repository's `BMS-Document.txt` contains infrastructure/setup material. This troubleshooting document intentionally does not reproduce tokens, passwords, access keys, or other secret values from that file.
