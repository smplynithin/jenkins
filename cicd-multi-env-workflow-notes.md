# CI/CD Workflow: Dev → Staging → Pre-Prod → Production
### Complete Notes with Branching Strategy, GitOps, and a Real Shared-Library Jenkins Pipeline

---

## 1. Why This Is Different From a Normal CI/CD Pipeline

Most CI/CD tutorials show **one pipeline deploying to one environment**. Real companies need the **same code to travel through multiple environments** before it reaches customers — and that journey is driven entirely by **branching strategy**, not by one giant pipeline.

The chain looks like:

```
Feature branch → Dev cluster
Main branch     → Staging cluster
Release branch  → Pre-prod cluster → Production cluster
```

Everything else in this document explains how that chain is actually built and triggered in real organizations.

---

## 2. Git Branching Strategy — Accurate Structure

This is a **GitFlow-style strategy** (the industry-standard model most companies use, in some simplified form). There are four branch types, each with a specific role, lifespan, and merge direction.

### 2.1 Branch Types & Rules

| Branch | Cut from | Merges into | Lifespan | Purpose |
|---|---|---|---|---|
| **`feature/*`** | `main` | `main` (via PR) | Temporary — deleted after merge | Isolated development of one feature/task. Zero, one, or many can exist at once. |
| **`main`** (a.k.a. `master`) | — (root branch) | — | Permanent — always exists | Single source of truth for **active, integrated development**. Always exactly one. |
| **`release/*`** | `main` | `main` (back-merge) + tagged for production | Temporary — lives through one release cycle, deleted/archived after | Cut when a set of features is "feature-complete" and ready to stabilize for shipping. Only bug fixes allowed here — **no new features**. |
| **`hotfix/*`** | The live production tag/`release` branch | `main` **+** the active `release` branch **+** production | Temporary — created only when needed | Fixes a **critical bug found in production** without waiting for the next full release cycle. |

### 2.2 The Critical Detail Most People Get Wrong

A **release branch is a stabilization branch, not a deploy-once branch.** Once cut from `main`:
- No new features are merged into it.
- Only bug fixes found during pre-prod/UAT testing are committed directly to it (or cherry-picked from `main`).
- When it's finally approved for production, it is **merged back into `main`** (so `main` gets any bugfixes that happened during stabilization) **and tagged** (e.g. `v2.4.0`) — that tag is what actually gets deployed to production.
- The release branch can stay alive for patch releases (`v2.4.1`, `v2.4.2`) if the team supports multiple live versions simultaneously — this is exactly the "iOS 14 / iOS 15 / iOS 16 all live at once" example.

A **hotfix branch always merges in three directions**, not one:
1. Into `main` — so the fix is never lost in future development.
2. Into the currently active `release/*` branch (if one is in-flight) — so the next release also has the fix.
3. Into production directly (or as a fast-tracked patch release/tag) — so customers get the fix immediately, without waiting for the normal release cycle.

### 2.3 Branch Flow Diagram

```mermaid
gitGraph
   commit id: "init"
   branch feature/login
   checkout feature/login
   commit id: "add login UI"
   commit id: "add validation"
   checkout main
   merge feature/login id: "merge login"
   branch release/v2.4
   checkout release/v2.4
   commit id: "fix UAT bug #1"
   checkout main
   commit id: "next feature starts"
   checkout release/v2.4
   commit id: "fix UAT bug #2" tag: "v2.4.0"
   checkout main
   merge release/v2.4 id: "back-merge release"
   checkout release/v2.4
   branch hotfix/critical-auth-bug
   checkout hotfix/critical-auth-bug
   commit id: "patch auth bug" tag: "v2.4.1"
   checkout main
   merge hotfix/critical-auth-bug id: "hotfix into main"
   checkout release/v2.4
   merge hotfix/critical-auth-bug id: "hotfix into release"
```

**Reading this diagram:**
- `feature/login` branches off `main`, does its work, merges back — standard PR flow.
- `release/v2.4` branches off `main` once features are ready to stabilize. Bug fixes land directly on it during UAT.
- `v2.4.0` tag = what goes to production first.
- `release/v2.4` **back-merges into `main`** so those UAT fixes aren't lost.
- A critical bug found *after* `v2.4.0` is live triggers `hotfix/critical-auth-bug`, cut from the release branch, tagged `v2.4.1`, and merged into **both** `main` and `release/v2.4`.

### 2.4 Interview Soundbite

> "We follow a GitFlow-style strategy with four branch types. Feature branches are short-lived and isolated, main is the single integration branch, release branches are stabilization-only branches cut when we're feature-complete, and hotfix branches handle critical production issues by branching off the live release and merging back into main, the active release branch, and production — never just one of those."

---

## 3. Branch → Environment Mapping

Each branch type triggers its **own webhook**, which fires its **own Jenkins pipeline**, which deploys to its **own Kubernetes cluster**:

```mermaid
flowchart LR
    A["Feature branch<br/>developer commits"] --> B["Dev cluster<br/>feature environment"]
    B --> C["Main branch<br/>merged, active dev"]
    C --> D["Staging cluster<br/>QA testing environment"]
    D --> E["Release branch<br/>cut after QA sign-off"]
    E --> F["Pre-prod cluster<br/>UAT, final verification"]
    F --> G["Production cluster<br/>live to customers"]
```

| Branch | Triggers | Deploys to | Owned/tested by |
|---|---|---|---|
| `feature/*` | Webhook → Jenkins pipeline (dev) | Dev cluster (feature-specific) | The developer working on that feature |
| `main` | Webhook → Jenkins pipeline (staging) | Staging / QA cluster | QA team — manual + automated testing |
| `release/*` | Webhook → Jenkins release pipeline | Pre-prod → then Production | Release managers, final verification |
| `hotfix/*` | Webhook → Jenkins hotfix pipeline (fast-tracked, fewer gates) | Production directly (patch) | On-call / release engineer |

**Key insight:** the pipelines are structurally similar (checkout → build → test → code quality → image build → scan → push → deploy) but the *later* pipelines (pre-prod/prod) often have **extra stages** — performance testing, DDoS simulation, penetration testing — because these take time and aren't worth running on every single commit to a feature branch.

---

## 4. The CI Stages (Common to Every Pipeline, Regardless of Environment)

1. **Code checkout** — pull the exact commit that triggered the build
2. **Build & unit test** — e.g., Maven for Java
3. **Code quality scan** — static analysis (SonarQube) — catches duplicate code, syntax/semantic issues
4. **Docker image build** — containerize the app
5. **Image scan** — Trivy / Snyk / Clair — check for critical vulnerabilities in the image
6. **Push to image registry** — Docker Hub / Quay / Artifactory / ECR

**Additional stages only at pre-prod/production level:**
- Performance testing
- DDoS/load resilience testing
- Penetration (security) testing

---

## 5. GitOps — How Real Companies Handle the Deployment Side

- **Keep application manifests in a separate Git repository** from the source code (a dedicated "manifest repo" / "GitOps repo") — not mixed with app source.
- Manifests are usually **Helm charts** rather than raw YAML, with a `values.yaml` per environment.
- Instead of the CI pipeline directly deploying, it **updates the image tag in the manifest repo** (e.g., `image.tag: v2.4.1`), commits, and pushes.
- **ArgoCD (or Flux)** watches that manifest repo and auto-syncs the change to the target Kubernetes cluster — this is the actual "CD" in GitOps.

### 5.1 The Real-Company Pattern

- Don't run a separate ArgoCD instance per environment.
- Run **one single ArgoCD instance** for the whole org.
- Structure the GitOps repo with **one folder per environment** (`dev/`, `staging/`, `pre-prod/`, `prod/`), each holding that environment's Helm chart/values.
- In ArgoCD, create **one "Application" per folder**, each pointing to a different **destination cluster**.

```mermaid
flowchart TB
    R["GitOps repository<br/>folders: dev / staging / pre-prod / prod"] --> A["ArgoCD<br/>single instance, watches all folders"]
    A --> D1["Dev cluster<br/>syncs dev/ folder"]
    A --> D2["Staging cluster<br/>syncs staging/ folder"]
    A --> D3["Pre-prod cluster<br/>syncs pre-prod/ folder"]
    A --> D4["Prod cluster<br/>syncs prod/ folder"]
```

**Why this matters in an interview:** it shows you understand separation of concerns — CI (Jenkins, many pipelines, one per environment/branch) is decoupled from CD (one ArgoCD instance, declarative, git-driven) — rather than Jenkins directly holding cluster credentials and running `kubectl` or `helm upgrade` itself.

---

## 6. Why Separate Kubernetes Clusters Per Environment

- **Best practice:** dedicated cluster per environment (dev, staging, pre-prod, prod).
- **Common shortcut some companies take:** shared cluster with separate *namespaces* for dev/staging — acceptable but not ideal.
- **Non-negotiable in most real orgs:** pre-prod and production must be **separate clusters** — production access is tightly restricted; QA/dev teams don't get access to it.
- **Purpose of pre-prod specifically:** it's a safe place to **reproduce and debug customer-reported production issues** without touching the live environment directly.

---

## 7. The QA/Staging Gate

Once code merges to `main` and deploys to staging:
1. Automated regression tests (already part of the Jenkins pipeline) run.
2. QA team performs **manual testing** — this can take days, not just minutes — including performance, DDoS resilience, and penetration testing.
3. **Only after QA sign-off** does a release engineer cut a `release/*` branch from `main`.
4. The release branch triggers the release pipeline → pre-prod → (after verification) → production.

---

## 8. Real-Time Production Jenkinsfile — Shared Library Approach

This is how real companies actually implement the branch-to-environment logic described above: **one shared library pipeline function, reused by every microservice repo**, with the environment/stage behavior driven entirely by which branch triggered the build. This avoids every team writing (and maintaining) their own 80-line Jenkinsfile.

### 8.1 Shared Library Repository Structure

```
jenkins-shared-library/            (separate Git repo, referenced via @Library)
├── vars/
│   ├── multiEnvPipeline.groovy    ← the main entry point, called from every app repo
│   ├── buildAndTest.groovy
│   ├── codeQualityGate.groovy
│   ├── dockerBuildAndScan.groovy
│   ├── pushToRegistry.groovy
│   └── updateGitOpsRepo.groovy
├── src/
│   └── org/company/EnvConfig.groovy   ← resolves cluster/namespace per branch type
└── resources/
    └── org/company/gitops-values-template.yaml
```

### 8.2 `src/org/company/EnvConfig.groovy` — Branch-to-Environment Resolver

```groovy
package org.company

class EnvConfig implements Serializable {

    static Map resolve(String branchName) {
        if (branchName.startsWith('feature/')) {
            return [
                env: 'dev',
                cluster: 'dev-cluster',
                gitopsFolder: 'dev',
                requiresApproval: false,
                runSecurityTests: false
            ]
        } else if (branchName == 'main') {
            return [
                env: 'staging',
                cluster: 'staging-cluster',
                gitopsFolder: 'staging',
                requiresApproval: false,
                runSecurityTests: false
            ]
        } else if (branchName.startsWith('release/')) {
            return [
                env: 'preprod-then-prod',
                cluster: 'preprod-cluster',   // pipeline promotes to prod-cluster after approval
                gitopsFolder: 'pre-prod',
                requiresApproval: true,
                runSecurityTests: true
            ]
        } else if (branchName.startsWith('hotfix/')) {
            return [
                env: 'prod',
                cluster: 'prod-cluster',
                gitopsFolder: 'prod',
                requiresApproval: true,
                runSecurityTests: true
            ]
        } else {
            // PR builds / misc branches - build & test only, no deploy
            return [
                env: 'none',
                cluster: null,
                gitopsFolder: null,
                requiresApproval: false,
                runSecurityTests: false
            ]
        }
    }
}
```

### 8.3 `vars/multiEnvPipeline.groovy` — The Reusable Pipeline

```groovy
import org.company.EnvConfig

def call(Map config = [:]) {

    def appName   = config.appName
    def chartPath = config.chartPath ?: "./charts/${appName}"

    pipeline {
        agent none

        options {
            timeout(time: 45, unit: 'MINUTES')
            timestamps()
            disableConcurrentBuilds()
            buildDiscarder(logRotator(numToKeepStr: '20'))
        }

        environment {
            IMAGE_NAME   = "myorg/${appName}"
            ECR_REGISTRY = '123456789.dkr.ecr.us-east-1.amazonaws.com'
            IMAGE_TAG    = "${env.GIT_COMMIT.take(7)}-${env.BUILD_NUMBER}"
            SONAR_TOKEN  = credentials('sonar-token-id')
        }

        stages {

            stage('Resolve Environment') {
                agent { label 'build' }
                steps {
                    script {
                        env.RESOLVED_ENV = EnvConfig.resolve(env.BRANCH_NAME).env
                        env.TARGET_CLUSTER = EnvConfig.resolve(env.BRANCH_NAME).cluster
                        env.GITOPS_FOLDER = EnvConfig.resolve(env.BRANCH_NAME).gitopsFolder
                        env.NEEDS_APPROVAL = EnvConfig.resolve(env.BRANCH_NAME).requiresApproval.toString()
                        env.RUN_SEC_TESTS = EnvConfig.resolve(env.BRANCH_NAME).runSecurityTests.toString()
                        echo "Branch ${env.BRANCH_NAME} resolved → env=${env.RESOLVED_ENV}, cluster=${env.TARGET_CLUSTER}"
                    }
                }
            }

            stage('Checkout') {
                agent { label 'build' }
                steps { checkout scm }
            }

            stage('Build & Unit Test') {
                agent { label 'build' }
                steps { buildAndTest() }              // calls vars/buildAndTest.groovy
            }

            stage('Code Quality Gate') {
                agent { label 'build' }
                steps { codeQualityGate(appName) }     // calls vars/codeQualityGate.groovy
            }

            stage('Docker Build & Trivy Scan') {
                agent { label 'docker' }
                steps {
                    dockerBuildAndScan(
                        image: "${IMAGE_NAME}:${IMAGE_TAG}"
                    )                                   // calls vars/dockerBuildAndScan.groovy
                }
            }

            stage('Push to ECR') {
                when { expression { env.RESOLVED_ENV != 'none' } }
                agent { label 'docker' }
                steps {
                    pushToRegistry(
                        image: "${IMAGE_NAME}:${IMAGE_TAG}",
                        registry: "${ECR_REGISTRY}"
                    )                                   // calls vars/pushToRegistry.groovy
                }
            }

            stage('Extended Security Tests') {
                when { expression { env.RUN_SEC_TESTS == 'true' } }
                agent { label 'build' }
                steps {
                    echo 'Running performance, DDoS resilience, and penetration tests...'
                    sh './scripts/run-perf-tests.sh'
                    sh './scripts/run-pen-tests.sh'
                }
                // Only runs for release/* and hotfix/* branches - too slow for every commit
            }

            stage('Approve Deployment') {
                when { expression { env.NEEDS_APPROVAL == 'true' } }
                steps {
                    input message: "Deploy ${appName} (${IMAGE_TAG}) to ${env.TARGET_CLUSTER}?",
                          ok: 'Deploy', submitter: 'release-managers'
                }
            }

            stage('Update GitOps Repo') {
                when { expression { env.RESOLVED_ENV != 'none' } }
                agent { label 'build' }
                steps {
                    updateGitOpsRepo(
                        appName: appName,
                        folder: env.GITOPS_FOLDER,
                        imageTag: env.IMAGE_TAG,
                        chartPath: chartPath
                    )                                   // calls vars/updateGitOpsRepo.groovy
                }
                // ArgoCD (single instance, watching this repo) auto-syncs to env.TARGET_CLUSTER
            }

            stage('Promote Release to Production') {
                // only relevant on release/* branches, AFTER pre-prod verification passes
                when {
                    allOf {
                        expression { env.BRANCH_NAME.startsWith('release/') }
                        expression { env.RESOLVED_ENV != 'none' }
                    }
                }
                steps {
                    input message: 'Pre-prod verified. Promote to PRODUCTION?',
                          ok: 'Promote', submitter: 'release-managers'
                    script {
                        updateGitOpsRepo(
                            appName: appName,
                            folder: 'prod',
                            imageTag: env.IMAGE_TAG,
                            chartPath: chartPath
                        )
                    }
                }
            }
        }

        post {
            always {
                node('build') { cleanWs() }
            }
            success {
                slackSend channel: '#ci-cd', color: 'good',
                    message: "✅ ${appName} ${IMAGE_TAG} → ${env.RESOLVED_ENV} succeeded (${env.BRANCH_NAME})"
            }
            failure {
                slackSend channel: '#ci-cd', color: 'danger',
                    message: "❌ ${appName} pipeline FAILED on ${env.BRANCH_NAME} — ${env.BUILD_URL}"
            }
        }
    }
}
```

### 8.4 `vars/updateGitOpsRepo.groovy` — Helper Called Above

```groovy
def call(Map config) {
    withCredentials([usernamePassword(
        credentialsId: 'gitops-repo-creds',
        usernameVariable: 'GIT_USER',
        passwordVariable: 'GIT_PASS'
    )]) {
        sh """
            git clone https://\$GIT_USER:\$GIT_PASS@github.com/myorg/gitops-repo.git
            cd gitops-repo
            yq -i '.image.tag = "${config.imageTag}"' apps/${config.appName}/values-${config.folder}.yaml
            git commit -am "Deploy ${config.appName} ${config.imageTag} to ${config.folder}"
            git push
        """
    }
}
```

### 8.5 The Application Repo's Jenkinsfile — What Every Team Actually Writes

This is the entire Jenkinsfile a microservice team needs — everything else lives in the shared library and is maintained centrally:

```groovy
@Library('jenkins-shared-library@v3.2.0') _

multiEnvPipeline(
    appName: 'payments-service'
)
```

That's it. Three lines. **This is the real-world payoff of shared libraries** — 30+ microservice repos can each have a 3-line Jenkinsfile, while all the actual pipeline logic, branch-to-environment mapping, security gates, and GitOps update logic is maintained in **one place**. A bug fix or new stage added to the shared library instantly applies to every repo using it (pinned to a version tag for safety, e.g. `@v3.2.0`, so it's not a silent breaking change).

### 8.6 Why This Matches the Branching Strategy Exactly

| Branch pushed | `EnvConfig.resolve()` returns | Pipeline behavior |
|---|---|---|
| `feature/checkout-fix` | `env: dev`, no approval, no security tests | Fast build → dev cluster, no gates |
| `main` | `env: staging`, no approval, no security tests | Build → staging cluster, QA takes over manually from here |
| `release/v2.4` | `env: preprod-then-prod`, approval required, security tests run | Full pipeline including perf/DDoS/pen tests, manual gate before pre-prod, second manual gate before prod promotion |
| `hotfix/critical-auth-bug` | `env: prod`, approval required, security tests run | Fast-tracked but still gated — skips the multi-day staging soak, goes through security tests and a manual approval directly to prod |

---

## 9. End-to-End Flow Summary

```
Developer commits → feature/* branch → Webhook → Jenkins CI (shared lib) → Dev cluster
        ↓ (PR review & merge)
main branch → Webhook → Jenkins CI (shared lib) → Staging cluster → QA manual + automated testing
        ↓ (QA sign-off)
release/* branch cut → Jenkins release pipeline (shared lib) → Pre-prod cluster → Final verification
        ↓ (release-manager approval)
Same release pipeline promotes → Production cluster (tagged, e.g. v2.4.0)
        ↓ (if critical bug found later)
hotfix/* branch cut from release/prod tag → fast-tracked pipeline → Production
        ↓ back-merged into
main branch + active release/* branch (fix is never lost)
```

All deployments — regardless of environment — go through the **same GitOps repo + single ArgoCD instance**, just targeting a different folder and a different destination cluster each time.

---

## 10. Interview Soundbites — Quick Recap

- **Branching:** "GitFlow-style — feature branches for isolated work, main as the integration branch, release branches for stabilization-only changes, hotfix branches that merge into main, the active release branch, and production simultaneously."
- **Environment mapping:** "Each branch type has its own webhook-triggered pipeline and its own target cluster — feature→dev, main→staging, release→pre-prod→prod."
- **GitOps:** "CI and CD are decoupled — Jenkins never touches the cluster directly, it updates a GitOps repo, and a single ArgoCD instance watching per-environment folders handles the actual sync."
- **Shared libraries:** "Instead of every microservice repo maintaining its own Jenkinsfile, we centralize pipeline logic in a shared library — each repo's Jenkinsfile is a 2-3 line call to a shared pipeline function, versioned and pinned for safety."
- **Clusters:** "Dedicated clusters per environment is the ideal; at minimum, pre-prod and production must never share a cluster due to access control."
