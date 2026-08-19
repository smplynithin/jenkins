# Jenkins — 3 YOE DevOps Engineer Interview Prep Guide

---

## 1. Jenkins — Overview

Jenkins is an open-source **automation server** used to build, test, and deploy code continuously (CI/CD). It's plugin-driven, extensible, and runs pipelines defined either through the UI or as code (Jenkinsfile).

**Key interview line:** "Jenkins orchestrates the pipeline — it doesn't do the actual build/test/deploy work itself, it delegates to tools (Maven, Docker, kubectl, etc.) via shell steps and plugins."

---

## 2. Jenkins Architecture

```
                ┌─────────────────────┐
                │   Jenkins Controller  │  (Master)
                │  - Web UI            │
                │  - Job scheduling    │
                │  - Plugin mgmt       │
                └──────────┬──────────┘
                           │ SSH / JNLP
        ┌──────────────────┼──────────────────┐
        │                  │                  │
   ┌────▼────┐        ┌────▼────┐        ┌────▼────┐
   │ Agent 1  │        │ Agent 2 │        │ Agent N │
   │ (static/ │        │ (K8s pod│        │ (Docker)│
   │  VM)     │        │ dynamic)│        │         │
   └──────────┘        └─────────┘        └─────────┘
```

### Responsibility of each component

| Component | Responsibility |
|---|---|
| **Controller (Master)** | Schedules builds, serves the web UI, stores job configs, dispatches work to agents, holds plugin/system config, manages credentials store |
| **Agent (Node/Slave)** | Actual machine (VM, container, pod) where build steps execute; only needs a JVM + agent connector |
| **Executor** | A slot on a node that can run one build/step at a time (see below) |
| **Plugins** | Extend Jenkins — SCM integration, notifications, pipeline DSL, cloud provisioning |
| **Jenkins Home (`JENKINS_HOME`)** | Stores job configs, build history, plugins, credentials (on controller) |
| **Queue** | Holds jobs waiting for a free executor |

---

## 3. Executor

- An **executor** is a slot on an agent (or controller) capable of running **one build/step at a time**.
- One agent can have multiple executors (parallel build capacity).
- Controller executors should be set to **0** in production (best practice) — controller should only orchestrate, never run untrusted build code.
- **Interview trap:** "If you have 1 agent with 2 executors, can 2 different jobs run simultaneously on it?" → Yes, as long as both jobs fit within that agent's resources.

---

## 4. Pipeline

A **Pipeline** is a suite of plugins supporting CI/CD workflows defined **as code** (Jenkinsfile), checked into SCM. It replaces manually-clicked Freestyle jobs with versioned, reviewable automation.

Types:
- **Declarative Pipeline** — structured, opinionated syntax (recommended)
- **Scripted Pipeline** — Groovy-based, flexible, imperative

---

## 5. Scripted vs Declarative Pipeline

| Aspect | Declarative | Scripted |
|---|---|---|
| Syntax | Structured (`pipeline { }` block) | Pure Groovy (`node { }` block) |
| Learning curve | Easier, more readable | Requires Groovy knowledge |
| Flexibility | Limited to defined DSL (extend via `script{}`) | Fully flexible — loops, conditionals natively |
| Validation | Linted before execution (fails fast on syntax errors) | Errors surface at runtime |
| Restart capability | Supports `restartFromStage` | No |
| Industry preference | **Preferred/standard today** | Legacy / complex edge cases |

**Declarative skeleton:**
```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps { echo 'Building...' }
        }
    }
}
```

**Scripted skeleton:**
```groovy
node {
    stage('Build') {
        echo 'Building...'
    }
}
```

**Interview answer:** "Declarative for 95% of pipelines — readability, validation, restart-from-stage. Drop into `script{}` blocks inside Declarative when I need Groovy logic (loops, dynamic stage generation)."

---

## 6. Jenkinsfile — Full Declarative Syntax & Explanation

```groovy
pipeline {
    agent any                          // WHERE it runs

    options {                          // pipeline-level behavior
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
        retry(2)
    }

    parameters {                       // user inputs at trigger time
        string(name: 'ENV', defaultValue: 'dev', description: 'Target environment')
        booleanParam(name: 'DEPLOY', defaultValue: false, description: 'Deploy after build?')
    }

    environment {                      // variables available to all stages
        IMAGE_NAME = 'myapp'
        SONAR_TOKEN = credentials('sonar-token-id')
    }

    triggers {                         // what starts the pipeline automatically
        pollSCM('H/5 * * * *')
    }

    stages {                           // the actual work, broken into logical steps
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Build') {
            steps { sh 'mvn clean package' }
        }
    }

    post {                             // cleanup / notification, always runs
        always { cleanWs() }
        success { echo 'Pipeline succeeded' }
        failure { echo 'Pipeline failed' }
    }
}
```

Each block explained individually below.

---

## 7. Agent

Defines **where** a pipeline or stage executes.

```groovy
agent any                 // any available agent
agent none                // no default agent; each stage must define its own
agent { label 'linux' }   // specific labeled node
agent { docker { image 'maven:3.9-eclipse-temurin-17' } }
agent { kubernetes { yaml podYamlString } }
```

- Can be set at **pipeline level** (applies to all stages) or **stage level** (override per stage).
- `agent none` + per-stage agents is common in production for mixed workloads (e.g., build on a Maven-labeled agent, deploy on a kubectl-labeled agent).

---

## 8. Environment — Types & Uses

```groovy
environment {
    GLOBAL_VAR = 'value'                       // global to entire pipeline
    DB_PASS = credentials('db-password-id')    // pulled securely from credentials store
}

stages {
    stage('Deploy') {
        environment {
            STAGE_VAR = 'only-here'            // scoped to this stage only
        }
        steps { sh 'echo $STAGE_VAR' }
    }
}
```

**Types:**
- **Pipeline-level (global)** — available to every stage
- **Stage-level** — overrides/adds vars only within that stage
- **Built-in env vars** — `BUILD_NUMBER`, `BUILD_URL`, `JOB_NAME`, `WORKSPACE`, `GIT_COMMIT`
- **Credentials-backed env vars** — auto-masked in console logs

**Use case:** storing image tags, registry URLs, non-secret config, and securely injecting secrets without echoing them in logs.

---

## 9. Parameters — Types & Uses

```groovy
parameters {
    string(name: 'BRANCH', defaultValue: 'main', description: '')
    choice(name: 'ENV', choices: ['dev','qa','prod'], description: '')
    booleanParam(name: 'RUN_TESTS', defaultValue: true, description: '')
    text(name: 'RELEASE_NOTES', defaultValue: '', description: '')
    password(name: 'API_KEY', defaultValue: '', description: '')
}
```

**Types:** `string`, `choice`, `booleanParam`, `text`, `password`, `file`

**Use case:** letting a human (or upstream job) decide environment, version, or feature flags at trigger time — e.g., choosing `dev`/`qa`/`prod` for the same pipeline.

---

## 10. Options — Types & Uses

```groovy
options {
    timeout(time: 1, unit: 'HOURS')        // kill pipeline if it hangs
    retry(3)                                // retry whole pipeline on failure
    timestamps()                            // add timestamps to console log
    disableConcurrentBuilds()               // prevent overlapping runs
    buildDiscarder(logRotator(numToKeepStr: '10'))  // retention policy
    skipDefaultCheckout()                   // don't auto-checkout SCM
    ansiColor('xterm')                      // colored console output
}
```

**Use case:** governance/safety rails — prevent runaway builds, control concurrency, manage log/artifact retention.

---

## 11. Stages — Components & Logic

```groovy
stages {
    stage('Build') {
        when { branch 'main' }             // conditional execution
        steps {
            sh 'mvn package'
        }
        post {
            success { echo 'Build ok' }
        }
    }
    stage('Parallel Tests') {
        parallel {
            stage('Unit') { steps { sh 'mvn test' } }
            stage('Integration') { steps { sh 'mvn verify' } }
        }
    }
}
```

**Components:**
- `steps` — actual commands/actions
- `when` — conditional logic (branch, environment, expression)
- `post` — stage-level hooks
- `parallel` — concurrent sub-stages
- `agent` — stage-specific execution node

**Logic flow:** stages run **sequentially top-to-bottom** by default; a failed stage stops the pipeline (unless wrapped in `catchError`/`try-catch`).

---

## 12. Credentials Store

Jenkins has a built-in encrypted **Credentials Store** (`Manage Jenkins → Credentials`).

**Types:**
- Username/Password
- SSH Username with Private Key
- Secret text (API tokens)
- Secret file (kubeconfig, .pem, etc.)
- Certificate

**Scopes:** Global, System, Folder-level (restrict which jobs can use which creds)

**Usage in Jenkinsfile:**
```groovy
environment {
    DOCKER_CREDS = credentials('docker-hub-creds')  // exposes _USR and _PSW
}
// or
withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
    sh 'docker login -u $USER -p $PASS'
}
```

Secrets are **automatically masked** as `****` in console output.

---

## 13. Triggers — Types & Explanation

```groovy
triggers {
    pollSCM('H/5 * * * *')        // poll repo every 5 min for changes
    cron('H 2 * * *')             // scheduled, time-based (nightly build)
    upstream(upstreamProjects: 'job-A', threshold: hudson.model.Result.SUCCESS)
    githubPush()                  // via webhook (GitHub plugin)
}
```

| Trigger Type | Explanation |
|---|---|
| **SCM Polling** | Jenkins checks repo periodically for changes — inefficient, adds delay, avoid in production |
| **Webhook** | Repo pushes an event to Jenkins instantly on commit/PR — **preferred production practice** |
| **Cron/Scheduled** | Time-based, for nightly builds, scheduled reports |
| **Upstream/Downstream** | Job B triggers automatically after Job A completes |
| **Manual** | Human clicks "Build Now" |

---

## 14. Webhook

A **webhook** is an HTTP callback: the SCM (GitHub/GitLab/Bitbucket) sends a POST request to Jenkins the moment an event happens (push, PR, merge), triggering the pipeline instantly.

**Why (production practice):**
- Near-instant trigger vs polling delay
- Reduces load on both Jenkins and the SCM server (no constant polling)
- Can carry payload data (branch, commit SHA, PR number) used for smarter pipeline logic
- Requires Jenkins to be reachable from the SCM (public URL/reverse proxy, or use a relay like smee.io for local testing)

**Setup:** Repo settings → Webhooks → Payload URL `http://<jenkins-url>/github-webhook/` → content type `application/json` → select push/PR events.

---

## 15. SCM (Source Control Management)

`checkout scm` — pulls source from the configured repository (Git/SVN/etc.) using branch/credentials defined in the job or Multibranch config.

```groovy
steps {
    checkout scm
    // or explicit:
    git branch: 'main', url: 'https://github.com/org/repo.git', credentialsId: 'git-creds'
}
```

---

## 16. Multibranch Pipeline & Its Structure

A **Multibranch Pipeline** auto-discovers branches (and PRs) in a repo and creates a Jenkins job **per branch** automatically, each running its own Jenkinsfile from that branch.

**Structure:**
```
Multibranch Pipeline Job
 ├── main         (from Jenkinsfile in main branch)
 ├── develop      (from Jenkinsfile in develop branch)
 ├── feature/xyz  (auto-created on branch push, auto-removed when branch deleted)
 └── PR-42        (if PR discovery enabled)
```

**Key points:**
- Each branch's pipeline behavior is fully independent — defined by that branch's own Jenkinsfile
- Uses **branch source** config (GitHub/GitLab org or repo) + credentials
- Supports `when { branch 'main' }` for branch-specific stage logic within a shared Jenkinsfile
- Auto-cleans jobs for deleted branches (configurable orphan strategy)

---

## 17. How Jenkins Determines Successful Stage Completion

- A stage/step is marked **SUCCESS** if its shell command / step returns **exit code 0**.
- Non-zero exit code → stage marked **FAILURE**, pipeline halts by default (unless `catchError`, `try/catch`, or `post{}` handles it).
- **UNSTABLE** — a special state (e.g., test failures reported but build didn't crash) — commonly set by test/quality plugins via `currentBuild.result = 'UNSTABLE'`.
- **ABORTED** — manually stopped or timed out.
- Jenkins tracks `currentBuild.result` and `currentBuild.currentResult` — pipeline logic can branch on these.

---

## 18. Post — Types & Use Case for Each

```groovy
post {
    always     { echo 'Runs no matter what' }
    success    { echo 'Only if pipeline/stage succeeded' }
    failure    { echo 'Only if pipeline/stage failed' }
    unstable   { echo 'Tests failed but build did not crash' }
    aborted    { echo 'Manually cancelled or timed out' }
    changed    { echo 'Runs only if result differs from previous run' }
    fixed      { echo 'Failed last time, succeeded this time' }
    regression { echo 'Succeeded last time, failed this time' }
    cleanup    { cleanWs() }   // always runs LAST, after all other post conditions
}
```

**Use cases:**
- `always` → workspace cleanup, archiving logs
- `success` → trigger deployment, notify success channel
- `failure` → alert on-call/Slack, rollback trigger
- `unstable` → notify QA about flaky/failing tests without blocking pipeline
- `changed`/`fixed`/`regression` → smart notifications (only alert when state changes, not every green build)

---

## 19. Build Stage — Example

```groovy
stage('Build') {
    steps {
        sh 'mvn clean package -DskipTests'
    }
}
```
(For Node.js: `npm ci && npm run build`; for Docker-native builds: `docker build -t $IMAGE_NAME:$BUILD_NUMBER .`)

---

## 20. Checkout Stage — Example

```groovy
stage('Checkout') {
    steps {
        checkout scm
        sh 'git log -1 --pretty=format:"%H"'   // capture commit SHA for tagging
    }
}
```

---

## 21. Test Stage — Example

```groovy
stage('Test') {
    steps {
        sh 'mvn test'
    }
    post {
        always {
            junit 'target/surefire-reports/*.xml'   // publish test results to Jenkins UI
        }
    }
}
```

---

## 22. Store Artifact Stage — Example

```groovy
stage('Archive Artifact') {
    steps {
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
    }
}
```
Production variant — pushing to Nexus/Artifactory instead of just archiving locally:
```groovy
stage('Publish to Nexus') {
    steps {
        nexusArtifactUploader artifacts: [[artifactId: 'myapp', file: 'target/app.jar', type: 'jar']],
            nexusUrl: 'nexus.company.com', repository: 'releases', credentialsId: 'nexus-creds'
    }
}
```

---

## 23. SonarQube Stage — Example & Interview Questions

```groovy
stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('SonarQubeServer') {
            sh 'mvn sonar:sonar -Dsonar.projectKey=myapp'
        }
    }
}
stage('Quality Gate') {
    steps {
        timeout(time: 5, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
        }
    }
}
```

**Interview questions:**
- Why is the Quality Gate check usually in a **separate stage** with a **timeout**? → SonarQube analysis is async; Jenkins must poll/wait for the webhook callback, so it needs a bounded wait to avoid hanging forever.
- What's the difference between `withSonarQubeEnv` and `waitForQualityGate`? → First runs the scan, second blocks the pipeline until Sonar server responds pass/fail.
- What happens if `abortPipeline: true` and quality gate fails? → Pipeline stops immediately, deployment stages never run.
- How do you configure the webhook so SonarQube can call back to Jenkins? → SonarQube server → Administration → Webhooks → point to `<jenkins-url>/sonarqube-webhook/`.

---

## 24. Quality Gate

A **Quality Gate** is a set of pass/fail conditions (code coverage %, bugs, vulnerabilities, code smells, duplication) defined in SonarQube. If code doesn't meet the threshold, the gate **fails**, and the pipeline can be configured to stop before deployment — enforcing code quality as a hard gate, not just a report.

---

## 25. Static vs Dynamic Code Analysis

| | Static Analysis | Dynamic Analysis |
|---|---|---|
| **When** | Without executing the code (source-level) | While the application is running |
| **Examples** | SonarQube, ESLint, Checkstyle, Semgrep, Trivy (config/IaC scanning) | DAST tools (OWASP ZAP, Burp Suite), load testing |
| **Finds** | Code smells, bugs, security patterns, vulnerable dependencies | Runtime vulnerabilities, memory leaks, actual exploitability |
| **Pipeline stage** | Usually right after Build, before packaging | Usually after Deploy to a test environment |

---

## 26. Trivy Stage — Example & Interview Questions

```groovy
stage('Trivy Scan') {
    steps {
        sh 'trivy image --severity HIGH,CRITICAL --exit-code 1 --format table $IMAGE_NAME:$BUILD_NUMBER'
    }
}
```
Fail build only on critical CVEs, but still generate a report:
```groovy
stage('Trivy Scan') {
    steps {
        sh 'trivy image --format json -o trivy-report.json $IMAGE_NAME:$BUILD_NUMBER || true'
        sh 'trivy image --severity CRITICAL --exit-code 1 $IMAGE_NAME:$BUILD_NUMBER'
    }
}
```

**Interview questions:**
- What does Trivy scan? → Container images, filesystems, Git repos, IaC files (Terraform/K8s manifests) for CVEs and misconfigurations.
- Why `--exit-code 1` only on `CRITICAL`/`HIGH`? → To avoid blocking every build on low-severity noise while still gating on real risk.
- Where does Trivy fit in the pipeline? → After image build, before pushing to registry.
- How do you avoid scanning the same base image repeatedly? → Trivy caches its vulnerability DB locally between scans.

---

## 27. ECR Push & Related Questions

```groovy
stage('Push to ECR') {
    steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
            sh '''
                aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $ECR_REGISTRY
                docker tag $IMAGE_NAME:$BUILD_NUMBER $ECR_REGISTRY/$IMAGE_NAME:$BUILD_NUMBER
                docker push $ECR_REGISTRY/$IMAGE_NAME:$BUILD_NUMBER
            '''
        }
    }
}
```

**Related questions:**
- How do you authenticate Jenkins to ECR securely? → IAM role on the agent (preferred, no static keys) or scoped AWS credentials in Jenkins credential store.
- Why tag with `$BUILD_NUMBER` or commit SHA instead of `latest`? → Immutable, traceable deployments; `latest` makes rollback and auditing impossible.
- How do you handle ECR repo lifecycle (old image cleanup)? → ECR lifecycle policies (expire untagged images / keep last N).

---

## 28. Cosign — Explanation & Example

**Cosign** (Sigstore project) signs and verifies container images cryptographically, proving the image came from your trusted pipeline and wasn't tampered with before deployment.

```groovy
stage('Sign Image') {
    steps {
        withCredentials([file(credentialsId: 'cosign-key', variable: 'COSIGN_KEY')]) {
            sh 'cosign sign --key $COSIGN_KEY $ECR_REGISTRY/$IMAGE_NAME:$BUILD_NUMBER'
        }
    }
}
```
Verification (typically done at deploy/admission-control time, e.g., via a Kubernetes admission policy):
```bash
cosign verify --key cosign.pub $ECR_REGISTRY/$IMAGE_NAME:$BUILD_NUMBER
```

**Why it matters (interview angle):** supply-chain security — ensures only images signed by your CI pipeline can be deployed, blocking tampered or unauthorized images even if someone gets registry push access.

---

## 29. Updating GitOps Repo

Common pattern in GitOps (ArgoCD/Flux) setups: Jenkins doesn't deploy directly — it updates a manifest/values file in a separate Git repo, and the GitOps controller syncs it to the cluster.

```groovy
stage('Update GitOps Repo') {
    steps {
        withCredentials([usernamePassword(credentialsId: 'gitops-repo-creds', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
            sh '''
                git clone https://$GIT_USER:$GIT_PASS@github.com/org/gitops-repo.git
                cd gitops-repo
                yq -i '.image.tag = "'$BUILD_NUMBER'"' apps/myapp/values.yaml
                git commit -am "Update myapp image to $BUILD_NUMBER"
                git push
            '''
        }
    }
}
```

**Interview angle:** decouples CI (build/test/scan) from CD (deploy) — ArgoCD/Flux pulls, Jenkins never needs cluster credentials directly, improving security posture.

---

## 30. Helm Lint and Template Check

```groovy
stage('Helm Validate') {
    steps {
        sh 'helm lint ./charts/myapp'
        sh 'helm template ./charts/myapp --values ./charts/myapp/values-${ENV}.yaml > rendered.yaml'
        sh 'kubeval rendered.yaml'   // or kubeconform, to validate against K8s schema
    }
}
```
- `helm lint` → catches chart syntax/best-practice issues
- `helm template` → renders manifests without applying, so you can inspect/validate output before it touches a cluster

---

## 31. Other Common Production Stages (often missed)

- **Docker Build & Tag** (immutable tag = commit SHA or build number)
- **Unit + Integration + Contract tests** as separate stages
- **Dependency/license scan** (e.g., `npm audit`, `OWASP Dependency-Check`, Snyk)
- **Image push with retry logic**
- **Deploy to staging + smoke test** before prod promotion
- **Manual approval gate** before prod (`input` step)
- **Canary / Blue-Green deploy stage**
- **Post-deploy health check** (curl `/health`, readiness probe check)
- **Rollback stage** (triggered from `post { failure {} }`)
- **Notification stage** (Slack/Teams/email on final status)
- **Tagging release in Git** (`git tag`) after successful prod deploy
- **Changelog/release notes generation**

---

## 32. Shared Libraries — Explanation, Example & Related Questions

A **Shared Library** is reusable Groovy pipeline code stored in a separate Git repo, imported into any Jenkinsfile — avoids copy-pasting the same stages across dozens of repos.

**Structure:**
```
(shared-library-repo)
├── vars/
│   └── buildAndPush.groovy      // global function callable as buildAndPush()
├── src/
│   └── org/company/Utils.groovy // reusable Groovy classes
└── resources/
    └── some-template.yaml
```

**`vars/buildAndPush.groovy`:**
```groovy
def call(String imageName, String tag) {
    sh "docker build -t ${imageName}:${tag} ."
    sh "docker push ${imageName}:${tag}"
}
```

**Usage in a project's Jenkinsfile:**
```groovy
@Library('my-shared-library') _
pipeline {
    agent any
    stages {
        stage('Build & Push') {
            steps {
                buildAndPush('myapp', env.BUILD_NUMBER)
            }
        }
    }
}
```

**Related interview questions:**
- Why use Shared Libraries? → DRY principle — standardize CI/CD steps org-wide, single place to fix bugs/update logic for all pipelines.
- Difference between `vars/` and `src/`? → `vars/` = simple global step functions; `src/` = full Groovy classes for complex reusable logic.
- How do you version a shared library? → Reference a specific branch/tag: `@Library('my-lib@v2.1.0') _` for stability, avoiding breaking changes hitting all pipelines at once.
- Can shared libraries be loaded dynamically? → Yes, via `library identifier: 'my-lib@main', retriever: modernSCM(...)` inside a `script{}` block.

---

## 33. Parallel Stage

```groovy
stage('Test Suite') {
    parallel {
        stage('Unit Tests') { steps { sh 'mvn test' } }
        stage('Lint') { steps { sh 'npm run lint' } }
        stage('Security Scan') { steps { sh 'trivy fs .' } }
    }
}
```
**Use case:** cuts pipeline duration by running independent stages concurrently (each on its own executor/agent).

---

## 34. Matrix Build

```groovy
matrix {
    axes {
        axis { name: 'PLATFORM'; values 'linux', 'windows' }
        axis { name: 'JDK'; values '11', '17' }
    }
    stages {
        stage('Build') {
            steps { echo "Building on ${PLATFORM} with JDK ${JDK}" }
        }
    }
}
```
**Use case:** run the same set of stages across a combination of variables (OS × language version × environment) — generates all combinations automatically instead of writing separate stages manually.

---

## 35. Input Steps & Approval Gate Types

```groovy
stage('Approve Prod Deploy') {
    steps {
        input message: 'Deploy to production?', ok: 'Deploy',
              submitter: 'release-managers'
    }
}
```
With parameters collected at approval time:
```groovy
input message: 'Confirm rollout', parameters: [choice(name: 'STRATEGY', choices: ['canary','blue-green'])]
```

**Types of approval gates:**
- **Simple input** (`input message:`) — pause and wait for a click
- **Scoped submitter** (`submitter:`) — restrict who can approve (RBAC)
- **Timed input** — wrapped in `timeout {}` to auto-abort if nobody approves
- **Parameterized input** — collect a decision (e.g., rollout strategy) at approval time

**Interview note:** an `input` step **holds an executor** unless carefully placed (best practice: put it in a stage with `agent none` or outside the block using an agent, to avoid blocking a node while waiting on a human).

---

## 36. Error Handling — Types & Significance

```groovy
// 1. try/catch (Scripted or inside script{})
script {
    try {
        sh 'mvn test'
    } catch (err) {
        echo "Tests failed: ${err}"
        currentBuild.result = 'UNSTABLE'
    }
}

// 2. catchError - continues pipeline but marks build status
catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
    sh './flaky-step.sh'
}

// 3. post { failure { } } - pipeline-level reaction
post {
    failure {
        slackSend message: "Build ${env.BUILD_NUMBER} failed"
    }
}

// 4. retry() - re-attempt transient failures
retry(3) {
    sh 'curl -f https://flaky-service/health'
}

// 5. timeout() - prevent hangs
timeout(time: 10, unit: 'MINUTES') {
    sh './long-running-step.sh'
}
```

**Significance:** production pipelines must **fail gracefully** — distinguish between "stop everything now" (security/compile failure) vs "record and continue" (flaky non-critical test), and always notify + clean up regardless of outcome.

---

## 37. Static VM Agents vs Kubernetes (Dynamic) Agents

| | Static VM Agent | Kubernetes Agent (dynamic) |
|---|---|---|
| **Provisioning** | Pre-created, always running | Spun up as a pod per build, destroyed after |
| **Resource usage** | Wastes resources when idle | Scales to zero — pay only during build |
| **Isolation** | Shared state risk between builds if not cleaned | Fully isolated, ephemeral, clean every time |
| **Setup** | SSH/JNLP connection, manual tool installs | Kubernetes plugin, pod templates define tools per build |
| **Scaling** | Manual (add more VMs) | Automatic (cluster autoscaler) |
| **Best for** | Legacy/stable toolchains, licensed software tied to a host | Modern cloud-native CI, variable load, multi-tenant Jenkins |

```groovy
agent {
    kubernetes {
        yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.9-eclipse-temurin-17
    command: ['cat']
    tty: true
"""
    }
}
```

---

## 38. Workspace Cleanup

```groovy
post {
    always {
        cleanWs()   // deletes workspace contents after build
    }
}
```
Or selectively:
```groovy
cleanWs(patterns: [[pattern: 'target/**', type: 'INCLUDE']])
```
**Why:** prevents disk exhaustion on agents, avoids stale artifacts leaking into the next build, keeps builds reproducible.

---

## 39. Workspace Significance

- The **workspace** is the directory on the agent where source code is checked out and build steps execute.
- Each job typically gets its own workspace (`$WORKSPACE` env var), named after the job (and branch, for multibranch).
- Concurrent builds of the same job on the same agent can conflict over workspace unless `disableConcurrentBuilds()` or unique workspace naming (`ws()` step) is used.
- Ephemeral in K8s agents (destroyed with the pod); persistent on static VM agents (must be manually cleaned).

---

## 40. Notifications

```groovy
post {
    success {
        slackSend channel: '#ci-cd', color: 'good', message: "✅ ${env.JOB_NAME} #${env.BUILD_NUMBER} succeeded"
    }
    failure {
        emailext to: 'team@company.com', subject: "Build Failed: ${env.JOB_NAME}",
                  body: "Check console: ${env.BUILD_URL}"
    }
}
```
**Channels commonly used:** Slack, MS Teams, Email, PagerDuty/Opsgenie (for prod failures).
**Best practice:** notify on `failure`/`unstable` always; notify on `success` only for prod deploys (avoid noise for every dev-branch build) — use `changed`/`fixed` post conditions to reduce alert fatigue.

---

## 41. Common & Scenario-Based Interview Questions

**Conceptual:**
1. Difference between Freestyle and Pipeline jobs?
2. What happens if the Jenkins controller crashes mid-build?
3. How does Jenkins mask secrets in console logs, and can that be bypassed accidentally (e.g., `sh 'echo $PASSWORD'`)?
4. What's the difference between `stash`/`unstash` and `archiveArtifacts`?
5. How do you handle secrets management beyond Jenkins' built-in store (Vault integration)?

**Scenario-based:**
1. "A pipeline is stuck at the `input` step for 2 days and blocking an agent — how do you fix it?" → Move `input` off the agent-holding stage, or set a `timeout`; manually abort and re-trigger if needed.
2. "How would you design a pipeline for a microservices repo where each service has its own Dockerfile and independent versioning?" → Multibranch/parameterized pipeline with shared library for common build logic, path-based change detection (`changeset` in `when`) to only build affected services.
3. "Your SonarQube quality gate keeps timing out — what could be wrong?" → Webhook from SonarQube not reaching Jenkins (network/firewall), incorrect webhook URL, or polling stage timeout too short for analysis size.
4. "How do you roll back a failed Kubernetes deployment triggered from Jenkins?" → If GitOps: revert the commit in the GitOps repo (ArgoCD auto-syncs back); if direct deploy: `helm rollback` or `kubectl rollout undo` in a dedicated rollback stage.
5. "How do you scale Jenkins for 50+ teams pushing dozens of builds a day?" → Kubernetes dynamic agents (scale-to-zero), controller running only orchestration (0 executors), shared libraries for standardization, folder-based RBAC and credential scoping.
6. "How do you prevent a leaked secret from a compromised Jenkinsfile PR?" → Credential scoping (folder-level, not global), branch protection + PR review on Jenkinsfile changes, `withCredentials` masking, avoid printing env vars broadly, restrict `agent` trust for PR builds from forks.
7. "Blue-green vs canary — how would you implement each via Jenkins + Helm/K8s?" → Blue-green: deploy new version alongside old under a separate service, switch traffic via ingress/service selector swap in one stage. Canary: gradually shift traffic percentage (via Istio/Argo Rollouts) across staged pipeline steps with automated health-check gates between increments.

---

*Good luck with the interview — this covers concept → syntax → real production stage → the "why" behind it, which is exactly the depth expected at 3 YOE.*
