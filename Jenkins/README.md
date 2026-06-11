# Jenkins — Complete Study Guide
### All Topics · Detailed Q&A · Master Cheatsheet
**2025 Edition — Comprehensive coverage with examples, commands & cheatsheet**

---

## Table of Contents
- [Jenkins Basics](#jenkins-basics)
- [Declarative Pipeline](#declarative-pipeline)
- [Scripted Pipeline](#scripted-pipeline)
- [Shared Libraries](#shared-libraries)
- [Multibranch Pipelines](#multibranch)
- [Webhook Triggers](#webhook-triggers)
- [Agents: Docker, Kubernetes & Cloud](#agents)
- [Credentials & Security](#credentials-security)
- [Post-Build Actions: Artifacts, JUnit, Coverage](#post-build)
- [Notifications: Slack & Email](#notifications)
- [Configuration as Code (JCasC)](#jcasc)
- [Jenkins HA & Backup](#ha-backup)
- [Essential Plugins](#plugins)
- [Master Cheatsheet](#master-cheatsheet)

---

## Jenkins Basics

### 🟢 Q1. What is Jenkins and what problem does it solve?

**Explanation:**
Jenkins is an open-source automation server used for CI/CD. It automates building, testing, and deploying software, enabling teams to detect failures early and deliver frequently. Jenkins is highly extensible via 1800+ plugins.

```bash
# Install Jenkins on Ubuntu
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Get initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# Jenkins home directory
/var/lib/jenkins/
  jobs/           # Job configs and workspace
  plugins/        # Installed plugins
  secrets/        # Credentials
  config.xml      # Main Jenkins config
  nodes/          # Agent node configs
  users/          # User accounts

# Jenkins CLI
java -jar jenkins-cli.jar -s http://localhost:8080/ help
java -jar jenkins-cli.jar -s http://localhost:8080/ \
  -auth admin:token list-jobs
java -jar jenkins-cli.jar -s http://localhost:8080/ \
  -auth admin:token build my-job --wait
```

---

### 🟢 Q2. What is the difference between Freestyle and Pipeline jobs?

```
Freestyle:
  - GUI-based configuration
  - Sequential only
  - Plugin-dependent features
  - Not version-controlled
  - Hard to reproduce/audit

Pipeline:
  - Code-based (Jenkinsfile)
  - Stored in source control (GitOps)
  - Parallel stages
  - Conditional logic
  - Reusable via Shared Libraries
  - Full visibility into stages
```

---

## Declarative Pipeline

### 🟢 Q3. What are Declarative vs Scripted Pipelines?

```groovy
// ===== DECLARATIVE (recommended) =====
// Structured, validated syntax, opinionated
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'make build'
            }
        }
    }
}

// ===== SCRIPTED (Groovy DSL) =====
// Full Groovy, flexible but complex
node {
    stage('Build') {
        sh 'make build'
    }
}
```

---

### 🟡 Q4. What is the complete structure of a Declarative Pipeline?

```groovy
pipeline {
    // ===== AGENT =====
    agent {
        kubernetes {
            yaml """
            apiVersion: v1
            kind: Pod
            spec:
              containers:
              - name: build
                image: node:20-alpine
                command: ['sleep', 'infinity']
            """
            defaultContainer 'build'
        }
    }

    // ===== OPTIONS =====
    options {
        timeout(time: 30, unit: 'MINUTES')
        retry(2)                             // Retry failed pipeline up to 2 times
        timestamps()                         // Add timestamps to console output
        disableConcurrentBuilds()            // No parallel runs of same job
        skipDefaultCheckout()                // Don't auto-checkout
        buildDiscarder(logRotator(numToKeepStr: '10'))
        ansiColor('xterm')                   // Colored output
    }

    // ===== PARAMETERS =====
    parameters {
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Docker image tag')
        choice(name: 'ENVIRONMENT', choices: ['staging', 'production'], description: 'Target env')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Run test suite')
        password(name: 'DB_PASSWORD', description: 'Database password (override)')
    }

    // ===== TRIGGERS =====
    triggers {
        cron('H 2 * * 1-5')                 // Nightly Mon-Fri
        pollSCM('H/5 * * * *')              // Poll SCM every 5 min
        githubPush()                         // Trigger on GitHub webhook
        upstream(upstreamProjects: 'infra-job', threshold: hudson.model.Result.SUCCESS)
    }

    // ===== ENVIRONMENT =====
    environment {
        APP_NAME    = 'my-app'
        REGISTRY    = 'registry.example.com'
        IMAGE       = "${REGISTRY}/${APP_NAME}"
        // Bind credentials
        DOCKER_CREDS = credentials('docker-registry')
        SONAR_TOKEN  = credentials('sonar-token')
        // DOCKER_CREDS_USR and DOCKER_CREDS_PSW auto-set
    }

    // ===== TOOLS =====
    tools {
        maven 'Maven 3.9'
        jdk 'JDK 17'
        nodejs 'Node 20'
    }

    stages {
        // ===== CHECKOUT =====
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                    env.IMAGE_TAG = params.IMAGE_TAG ?: env.GIT_COMMIT_SHORT
                }
            }
        }

        // ===== PARALLEL STAGES =====
        stage('Quality Gates') {
            parallel {
                stage('Lint') {
                    steps {
                        sh 'npm run lint'
                    }
                }
                stage('Type Check') {
                    steps {
                        sh 'npm run typecheck'
                    }
                }
                stage('Security Scan') {
                    steps {
                        sh 'npm audit --audit-level moderate'
                    }
                }
            }
        }

        // ===== CONDITIONAL STAGE =====
        stage('Test') {
            when {
                expression { params.RUN_TESTS == true }
            }
            steps {
                sh 'npm test -- --coverage'
            }
            post {
                always {
                    junit 'test-results/**/*.xml'
                    publishHTML([
                        allowMissing: false,
                        reportDir: 'coverage',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }

        stage('Build & Push') {
            steps {
                script {
                    docker.withRegistry("https://${REGISTRY}", 'docker-registry') {
                        def image = docker.build("${IMAGE}:${env.IMAGE_TAG}")
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }

        stage('Deploy Staging') {
            when {
                branch 'main'
            }
            steps {
                withKubeConfig([credentialsId: 'staging-kubeconfig']) {
                    sh """
                        helm upgrade --install ${APP_NAME} ./charts/${APP_NAME} \
                          --namespace staging \
                          --set image.tag=${env.IMAGE_TAG} \
                          --atomic --wait
                    """
                }
            }
        }

        stage('Deploy Production') {
            when {
                allOf {
                    branch 'main'
                    expression { params.ENVIRONMENT == 'production' }
                }
            }
            input {
                message "Deploy to production?"
                ok "Deploy"
                parameters {
                    string(name: 'CONFIRM', description: 'Type YES to confirm')
                }
            }
            steps {
                withKubeConfig([credentialsId: 'prod-kubeconfig']) {
                    sh "helm upgrade --install ${APP_NAME} ./charts/${APP_NAME} --namespace production --set image.tag=${env.IMAGE_TAG} --atomic"
                }
            }
        }
    }

    // ===== POST =====
    post {
        always {
            cleanWs()
            archiveArtifacts artifacts: 'dist/**', allowEmptyArchive: true
        }
        success {
            slackSend(
                channel: '#deployments',
                color: 'good',
                message: "✅ ${APP_NAME} deployed successfully - ${env.IMAGE_TAG}"
            )
        }
        failure {
            slackSend(
                channel: '#deployments',
                color: 'danger',
                message: "❌ ${APP_NAME} pipeline FAILED - ${env.BUILD_URL}"
            )
            emailext(
                to: 'team@example.com',
                subject: "FAILED: ${currentBuild.fullDisplayName}",
                body: "Build URL: ${env.BUILD_URL}"
            )
        }
        unstable {
            slackSend(channel: '#deployments', color: 'warning', message: "⚠️ Tests unstable")
        }
    }
}
```

---

### 🟡 Q5. How do you handle when conditions?

```groovy
when {
    branch 'main'                          // Only on main branch
}
when {
    branch pattern: 'release/*', comparator: 'GLOB'
}
when {
    tag "v[0-9]*"                         // On version tags
}
when {
    environment name: 'DEPLOY', value: 'true'
}
when {
    expression { return currentBuild.number > 5 }
}
when {
    changeset "**/*.java"                  // Files changed match pattern
}
when {
    changelog '.*\\[RELEASE\\].*'         // Commit message matches
}
when {
    triggeredBy 'TimerTrigger'            // Only when triggered by cron
}
// Logical operators
when {
    allOf {
        branch 'main'
        expression { params.DEPLOY == 'true' }
    }
}
when {
    anyOf {
        branch 'main'
        branch 'release/*'
    }
}
when {
    not { branch 'development' }
}
// beforeAgent: true — evaluate when BEFORE allocating agent (saves resources)
when {
    beforeAgent true
    branch 'main'
}
```

---

### 🟡 Q6. How do you run parallel stages and matrix builds?

```groovy
// ===== PARALLEL STAGES =====
stage('Tests') {
    parallel {
        stage('Unit') {
            agent { label 'linux' }
            steps { sh 'npm run test:unit' }
        }
        stage('Integration') {
            agent { label 'linux' }
            steps { sh 'npm run test:integration' }
        }
        stage('E2E') {
            agent { label 'linux' }
            steps { sh 'npm run test:e2e' }
        }
    }
}

// ===== MATRIX BUILDS =====
stage('Cross-platform Build') {
    matrix {
        axes {
            axis {
                name 'PLATFORM'
                values 'linux', 'windows', 'macos'
            }
            axis {
                name 'NODE_VERSION'
                values '18', '20', '22'
            }
        }
        excludes {
            exclude {
                axis { name 'PLATFORM'; values 'macos' }
                axis { name 'NODE_VERSION'; values '18' }
            }
        }
        agent {
            label "${PLATFORM}"
        }
        stages {
            stage('Build') {
                tools { nodejs "node-${NODE_VERSION}" }
                steps {
                    sh 'npm install && npm run build'
                }
            }
            stage('Test') {
                steps { sh 'npm test' }
            }
        }
    }
}
```

---

## Shared Libraries

### 🔴 Q7. What are Shared Libraries and how do you create them?

```groovy
// Library structure:
// (root)
// ├── vars/                    # Global variables — pipeline steps
// │   ├── buildDockerImage.groovy
// │   ├── deployToKubernetes.groovy
// │   └── sendSlackNotification.groovy
// ├── src/                     # Groovy classes
// │   └── com/example/
// │       ├── Docker.groovy
// │       └── Kubernetes.groovy
// └── resources/               # Non-Groovy files
//     └── deploy-template.yaml

// vars/buildDockerImage.groovy
def call(Map config = [:]) {
    def registry = config.registry ?: 'registry.example.com'
    def imageName = config.image ?: error('image name required')
    def tag = config.tag ?: 'latest'
    def dockerfile = config.dockerfile ?: 'Dockerfile'

    docker.withRegistry("https://${registry}", config.credentialsId ?: 'docker-creds') {
        def image = docker.build("${registry}/${imageName}:${tag}", "-f ${dockerfile} .")
        image.push()
        if (config.latestTag) { image.push('latest') }
        return image
    }
}

// vars/deployToKubernetes.groovy
def call(Map config = [:]) {
    def namespace = config.namespace ?: 'default'
    def chart = config.chart ?: error('chart required')
    def release = config.release ?: error('release required')
    def values = config.values ?: []
    def kubeConfig = config.kubeConfig ?: 'kubeconfig'

    withKubeConfig([credentialsId: kubeConfig]) {
        def valuesArgs = values.collect { "-f ${it}" }.join(' ')
        sh """
            helm upgrade --install ${release} ${chart} \
              --namespace ${namespace} \
              --create-namespace \
              ${valuesArgs} \
              --set image.tag=${config.imageTag} \
              --atomic --wait --timeout 10m
        """
    }
}

// vars/sendSlackNotification.groovy
def call(String status, String channel = '#ci-cd') {
    def colors = [success: 'good', failure: 'danger', unstable: 'warning']
    def icons  = [success: '✅', failure: '❌', unstable: '⚠️']
    slackSend(
        channel: channel,
        color: colors[status] ?: 'grey',
        message: "${icons[status] ?: 'ℹ️'} *${env.JOB_NAME}* #${env.BUILD_NUMBER} - ${status.toUpperCase()}\n${env.BUILD_URL}"
    )
}

// src/com/example/Docker.groovy
package com.example

class Docker implements Serializable {
    def script
    String registry
    String credentialsId

    Docker(script, String registry, String credentialsId) {
        this.script = script
        this.registry = registry
        this.credentialsId = credentialsId
    }

    def buildAndPush(String name, String tag, String dockerfile = 'Dockerfile') {
        script.docker.withRegistry("https://${registry}", credentialsId) {
            def img = script.docker.build("${registry}/${name}:${tag}", "-f ${dockerfile} .")
            img.push()
            return img
        }
    }
}
```

```groovy
// Jenkinsfile using the shared library
@Library('my-shared-lib@main') _

pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                script {
                    buildDockerImage(
                        registry: 'registry.example.com',
                        image: 'my-app',
                        tag: env.GIT_COMMIT[0..7],
                        latestTag: true
                    )
                }
            }
        }
        stage('Deploy') {
            steps {
                deployToKubernetes(
                    release: 'my-app',
                    chart: './charts/my-app',
                    namespace: 'production',
                    values: ['values-prod.yaml'],
                    imageTag: env.GIT_COMMIT[0..7]
                )
            }
        }
        stage('Notify') {
            steps {
                sendSlackNotification('success', '#deployments')
            }
        }
    }
}
```

---

## Multibranch Pipelines

### 🟡 Q8. How do you set up Multibranch Pipelines?

**Explanation:**
Multibranch Pipeline automatically discovers branches/PRs in a repo and creates a job for each one. Each branch uses the `Jenkinsfile` in that branch. Jobs are auto-created and deleted as branches appear/disappear.

```groovy
// In Jenkins UI: New Item → Multibranch Pipeline
// Or via JCasC:
jobs:
  - script: |
      multibranchPipelineJob('my-app') {
        branchSources {
          github {
            id('my-app-github')
            scanCredentialsId('github-token')
            repoOwner('myorg')
            repository('my-app')
            traits {
              gitHubBranchDiscovery {
                strategyId(1)          // Exclude PRs merged to current
              }
              gitHubPullRequestDiscovery {
                strategyId(1)          // Merge with target branch
              }
              // Only build specific branches
              headWildcardFilter {
                includes('main release/* feature/*')
                excludes('')
              }
            }
          }
        }
        orphanedItemStrategy {
          discardOldItems {
            numToKeep(5)             // Keep last 5 builds of dead branches
          }
        }
        triggers {
          periodicFolderTrigger {
            interval('1d')           // Rescan repo daily
          }
        }
      }
```

```groovy
// Jenkinsfile — branch-aware logic
pipeline {
    agent any
    stages {
        stage('Build') {
            steps { sh 'make build' }
        }
        stage('Deploy Staging') {
            when { branch 'main' }
            steps { sh 'make deploy-staging' }
        }
        stage('Deploy Prod') {
            when { tag pattern: 'v\\d+\\.\\d+\\.\\d+', comparator: 'REGEXP' }
            steps { sh 'make deploy-prod' }
        }
        stage('PR Check') {
            when { changeRequest() }             // Only for PRs
            steps { sh 'make integration-test' }
        }
    }
}
```

---

## Webhook Triggers

### 🟡 Q9. How do you configure webhook triggers?

```groovy
// ===== GITHUB WEBHOOK =====
// 1. In Jenkins: Manage Jenkins → Configure System → GitHub → Advanced → "Secret text"
// 2. In GitHub repo: Settings → Webhooks → Add webhook
//    Payload URL: https://jenkins.example.com/github-webhook/
//    Content type: application/json
//    Events: Push, Pull Request

// In Jenkinsfile:
pipeline {
    triggers {
        githubPush()                 // Trigger on GitHub push
    }
    ...
}

// ===== GITLAB WEBHOOK =====
pipeline {
    triggers {
        gitlab(
            triggerOnPush: true,
            triggerOnMergeRequest: true,
            branchFilterType: 'All',
            secretToken: "${env.GITLAB_WEBHOOK_TOKEN}"
        )
    }
    ...
}

// ===== GENERIC WEBHOOK TRIGGER (flexible) =====
// Plugin: Generic Webhook Trigger
pipeline {
    triggers {
        GenericTrigger(
            genericVariables: [
                [key: 'BRANCH', value: '$.ref', regexpFilter: 'refs/heads/(.*)'],
                [key: 'COMMIT', value: '$.after'],
                [key: 'REPO', value: '$.repository.name']
            ],
            token: 'my-secret-token',       // Webhook URL: /generic-webhook-trigger/invoke?token=my-secret-token
            causeString: 'Triggered by push to $BRANCH',
            printContributedVariables: true,
            regexpFilterText: '$BRANCH',
            regexpFilterExpression: 'main|release/.*'  // Only trigger for these branches
        )
    }
    ...
}

// Webhook URL format:
// http://jenkins.example.com/generic-webhook-trigger/invoke?token=my-secret-token

// ===== BITBUCKET WEBHOOK =====
pipeline {
    triggers {
        bitbucketPush()
    }
    ...
}
```

---

## Agents: Docker, Kubernetes & Cloud

### 🟡 Q10. How do you configure Jenkins agents?

```groovy
// ===== DOCKER AGENT =====
pipeline {
    agent {
        docker {
            image 'node:20-alpine'
            args '-v /tmp:/tmp --network host'
            registryUrl 'https://registry.example.com'
            registryCredentialsId 'docker-creds'
        }
    }
    ...
}

// Per-stage docker agents
stage('Build') {
    agent {
        docker { image 'maven:3.9-openjdk-17' }
    }
    steps { sh 'mvn package' }
}

// ===== KUBERNETES AGENT (Jenkins Kubernetes Plugin) =====
pipeline {
    agent {
        kubernetes {
            inheritFrom 'default'
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins/agent: "true"
spec:
  serviceAccountName: jenkins-agent
  containers:
    - name: jnlp
      image: jenkins/inbound-agent:latest
      resources:
        requests: { cpu: 100m, memory: 256Mi }

    - name: build
      image: node:20-alpine
      command: [sleep]
      args: [infinity]
      resources:
        requests: { cpu: 500m, memory: 1Gi }
        limits: { cpu: 2, memory: 2Gi }

    - name: docker
      image: docker:24-dind
      securityContext:
        privileged: true
      env:
        - name: DOCKER_TLS_CERTDIR
          value: ""
      volumeMounts:
        - name: docker-sock
          mountPath: /var/run/docker.sock

    - name: helm
      image: alpine/helm:3.14.0
      command: [sleep]
      args: [infinity]

  volumes:
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock
"""
            defaultContainer 'build'
            podRetention onFailure()
        }
    }
    stages {
        stage('Build') {
            steps {
                container('build') {
                    sh 'npm install && npm run build'
                }
            }
        }
        stage('Docker Build') {
            steps {
                container('docker') {
                    sh 'docker build -t myapp:latest .'
                }
            }
        }
        stage('Deploy') {
            steps {
                container('helm') {
                    sh 'helm upgrade --install myapp ./chart'
                }
            }
        }
    }
}

// ===== EC2 CLOUD AGENT =====
// Jenkins → Manage Jenkins → Nodes and Clouds → New Cloud → EC2
// Config via JCasC:
jenkins:
  clouds:
    - amazonEC2:
        name: AWS
        region: us-east-1
        credentialsId: aws-credentials
        templates:
          - ami: ami-0abcdef1234567890
            instanceType: t3.medium
            labels: linux-build
            numExecutors: 2
            idleTerminationMinutes: 30
            maxTotalUses: 50
```

---

## Credentials & Security

### 🟡 Q11. How do you manage credentials in Jenkins?

```groovy
// Types of credentials:
// - Username + Password
// - Secret text
// - SSH Username with private key
// - Certificate
// - Secret file

// Using credentials in pipeline:
pipeline {
    environment {
        // Bind username/password — creates _USR and _PSW variables
        DOCKER_CREDS = credentials('docker-hub')
        // DOCKER_CREDS_USR = username
        // DOCKER_CREDS_PSW = password

        // Bind secret text
        API_TOKEN = credentials('my-api-token')
    }
    stages {
        stage('Push') {
            steps {
                sh "docker login -u ${DOCKER_CREDS_USR} -p ${DOCKER_CREDS_PSW}"
                // credentials are masked in logs with ****
            }
        }
    }
}

// Using withCredentials block (more flexible)
steps {
    withCredentials([
        usernamePassword(
            credentialsId: 'docker-hub',
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASS'
        ),
        string(
            credentialsId: 'sonar-token',
            variable: 'SONAR_TOKEN'
        ),
        sshUserPrivateKey(
            credentialsId: 'deploy-key',
            keyFileVariable: 'SSH_KEY_FILE',
            usernameVariable: 'SSH_USER'
        ),
        file(
            credentialsId: 'kubeconfig',
            variable: 'KUBECONFIG_FILE'
        )
    ]) {
        sh """
            docker login -u ${DOCKER_USER} -p ${DOCKER_PASS}
            export KUBECONFIG=${KUBECONFIG_FILE}
            kubectl apply -f deploy.yaml
        """
    }
}
```

---

### 🔴 Q12. What are Jenkins security best practices?

```yaml
# Jenkins hardening checklist:

# 1. Enable security
jenkins:
  securityRealm:
    ldap:
      configurations:
        - server: ldap://ldap.example.com
          rootDN: dc=example,dc=com
          userSearchBase: ou=users
          userSearch: uid={0}
  authorizationStrategy:
    roleBased:
      roles:
        global:
          - name: admin
            pattern: ".*"
            permissions:
              - "Overall/Administer"
          - name: developer
            permissions:
              - "Overall/Read"
              - "Job/Build"
              - "Job/Read"

# 2. CSRF protection (enabled by default in modern Jenkins)
# 3. Content Security Policy
# 4. Restrict script approval for Groovy scripts
# 5. Use credential binding instead of environment variables
# 6. Agent → Controller isolation (don't run jobs on controller)
# 7. Regular plugin updates
# 8. Audit logging
```

---

## Post-Build Actions

### 🟡 Q13. How do you archive artifacts, publish JUnit results, and code coverage?

```groovy
post {
    always {
        // ===== ARCHIVE ARTIFACTS =====
        archiveArtifacts(
            artifacts: 'dist/**/*.jar, dist/**/*.war, reports/**',
            allowEmptyArchive: true,
            fingerprint: true,           // Track which builds used this artifact
            onlyIfSuccessful: false
        )

        // ===== JUNIT TEST RESULTS =====
        junit(
            testResults: '**/test-results/**/*.xml, **/surefire-reports/**/*.xml',
            allowEmptyResults: true,
            skipPublishingChecks: false  // Publish to GitHub Checks API
        )

        // ===== COVERAGE REPORT (HTML Publisher) =====
        publishHTML([
            allowMissing: false,
            alwaysLinkToLastBuild: true,
            keepAll: true,
            reportDir: 'coverage/lcov-report',
            reportFiles: 'index.html',
            reportName: 'Code Coverage',
            reportTitles: 'Coverage'
        ])

        // ===== COBERTURA COVERAGE =====
        cobertura(
            coberturaReportFile: '**/coverage.xml',
            conditionalCoverageTargets: '70, 0, 0',
            lineCoverageTargets: '80, 0, 0',
            methodCoverageTargets: '80, 0, 0',
            failNoReports: false
        )

        // ===== JACOCO COVERAGE (Java) =====
        jacoco(
            execPattern: '**/**.exec',
            classPattern: '**/classes',
            sourcePattern: '**/src/main/java',
            minimumInstructionCoverage: '70',
            minimumBranchCoverage: '70'
        )

        // ===== PERFORMANCE RESULTS =====
        perfReport('**/jmeter-results.jtl')

        // ===== SONARQUBE =====
        withSonarQubeEnv('SonarQube') {
            sh 'mvn sonar:sonar'
        }
        // Wait for quality gate
        timeout(time: 5, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
        }

        // ===== STASH / UNSTASH between stages =====
        stash(name: 'build-artifacts', includes: 'dist/**')
    }
}

// Later stage:
unstash 'build-artifacts'

// ===== FINGERPRINTING =====
// Track file usage across builds
fingerprint 'dist/**/*.jar'
```

---

## Notifications

### 🟡 Q14. How do you configure Slack and email notifications?

```groovy
// ===== SLACK NOTIFICATIONS =====
// Plugin: Slack Notification
// Configure: Jenkins → Manage → Configure System → Slack
// - Workspace: myworkspace
// - Credential: slack-bot-token

// Basic
slackSend(
    channel: '#ci-cd',
    color: 'good',           // good / warning / danger / hex #FF0000
    message: "Build passed!"
)

// Rich message with blocks
slackSend(
    channel: '#deployments',
    blocks: [
        [type: "section", text: [type: "mrkdwn",
            text: "*${env.JOB_NAME}* build #${env.BUILD_NUMBER}"]],
        [type: "section", fields: [
            [type: "mrkdwn", text: "*Status:*\n✅ Success"],
            [type: "mrkdwn", text: "*Duration:*\n${currentBuild.durationString}"],
            [type: "mrkdwn", text: "*Branch:*\n${env.BRANCH_NAME}"],
            [type: "mrkdwn", text: "*Commit:*\n${env.GIT_COMMIT[0..7]}"]
        ]],
        [type: "actions", elements: [
            [type: "button", text: [type: "plain_text", text: "View Build"],
             url: env.BUILD_URL]
        ]]
    ]
)

// ===== EMAIL NOTIFICATIONS =====
// Plugin: Email Extension
// Configure: Jenkins → Manage → Configure System → Extended E-mail Notification

emailext(
    to: 'team@example.com',
    subject: "[Jenkins] ${currentBuild.result}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
    body: """
        <html>
        <body>
        <h2>Build ${currentBuild.result}</h2>
        <p><b>Job:</b> ${env.JOB_NAME}</p>
        <p><b>Build:</b> #${env.BUILD_NUMBER}</p>
        <p><b>Branch:</b> ${env.BRANCH_NAME}</p>
        <p><b>Duration:</b> ${currentBuild.durationString}</p>
        <p><b>URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
        ${currentBuild.result == 'FAILURE' ? '<p><b>Console Output:</b><br/>' + currentBuild.rawBuild.getLog(50).join('<br/>') + '</p>' : ''}
        </body>
        </html>
    """,
    mimeType: 'text/html',
    attachLog: currentBuild.result == 'FAILURE',
    compressLog: true,
    recipientProviders: [
        [$class: 'CulpritsRecipientProvider'],    // Committers who broke the build
        [$class: 'RequesterRecipientProvider']     // Person who triggered it
    ]
)
```

---

## Configuration as Code (JCasC)

### 🔴 Q15. What is Jenkins Configuration as Code (JCasC)?

**Explanation:**
JCasC allows the entire Jenkins configuration to be defined in YAML files, stored in source control. This enables reproducible, version-controlled Jenkins setups — essential for GitOps and disaster recovery.

```yaml
# jenkins.yaml — JCasC configuration file
# Place at: $JENKINS_HOME/casc_configs/jenkins.yaml
# Or set CASC_JENKINS_CONFIG env var

jenkins:
  systemMessage: "Jenkins managed by Configuration as Code"

  # Security realm (authentication)
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: admin
          password: "${JENKINS_ADMIN_PASSWORD}"  # From env var
        - id: deployer
          password: "${DEPLOYER_PASSWORD}"

  # Authorization
  authorizationStrategy:
    roleBased:
      roles:
        global:
          - name: admin
            permissions:
              - "Overall/Administer"
            assignments:
              - admin
          - name: developer
            permissions:
              - "Overall/Read"
              - "Job/Build"
              - "Job/Read"
              - "Job/Workspace"
            assignments:
              - deployer

  # Number of executors on controller (set 0 for controller-only)
  numExecutors: 0

  # Agent protocols
  agentProtocols:
    - JNLP4-connect

  # Nodes / agents
  nodes:
    - permanent:
        name: linux-builder-01
        labelString: linux build
        numExecutors: 4
        remoteFS: /var/jenkins
        launcher:
          ssh:
            host: 10.0.0.10
            credentialsId: ssh-agent-key
            javaPath: /usr/bin/java

credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              scope: GLOBAL
              id: docker-hub
              username: myuser
              password: "${DOCKER_PASSWORD}"
          - string:
              scope: GLOBAL
              id: sonar-token
              secret: "${SONAR_TOKEN}"
          - basicSSHUserPrivateKey:
              scope: GLOBAL
              id: deploy-ssh-key
              username: deploy
              privateKeySource:
                directEntry:
                  privateKey: "${SSH_PRIVATE_KEY}"

tool:
  maven:
    installations:
      - name: Maven 3.9
        home: /usr/share/maven
  jdk:
    installations:
      - name: JDK 17
        home: /usr/lib/jvm/java-17-openjdk-amd64
  git:
    installations:
      - name: Default
        home: /usr/bin/git

unclassified:
  location:
    url: https://jenkins.example.com/
    adminAddress: jenkins-admin@example.com

  # Slack config
  slackNotifier:
    teamDomain: myworkspace
    tokenCredentialId: slack-token
    room: '#ci-cd'

  # GitHub config
  githubpluginconfig:
    configs:
      - name: GitHub
        apiUrl: https://api.github.com
        credentialsId: github-token
        manageHooks: true

  # Email
  mailer:
    smtpHost: smtp.gmail.com
    smtpPort: "587"
    useSsl: false
    useTls: true
    authentication:
      username: jenkins@example.com
      password: "${SMTP_PASSWORD}"

jobs:
  - script: |
      folder('microservices')
  - script: |
      multibranchPipelineJob('microservices/api-service') {
        branchSources {
          github {
            id('api-service')
            repoOwner('myorg')
            repository('api-service')
            credentialsId('github-token')
          }
        }
      }
```

```bash
# Apply JCasC config
# Jenkins will auto-apply on startup if CASC_JENKINS_CONFIG is set

# Reload config without restart (via UI or API)
curl -X POST http://admin:token@jenkins:8080/configuration-as-code/reload

# Export current config
curl http://admin:token@jenkins:8080/configuration-as-code/export > current-jenkins.yaml

# Check JCasC plugin status
curl http://admin:token@jenkins:8080/configuration-as-code/check
```

---

## Jenkins HA & Backup

### 🔴 Q16. How do you set up Jenkins HA and backup?

```bash
# ===== JENKINS HOME BACKUP =====
#!/bin/bash
# backup-jenkins.sh

JENKINS_HOME=/var/lib/jenkins
BACKUP_DIR=/backups/jenkins
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/jenkins_${DATE}.tar.gz"

# Create backup directory
mkdir -p $BACKUP_DIR

# Stop Jenkins for consistent backup (or use thin-backup plugin)
# systemctl stop jenkins

# Backup important directories
tar -czf $BACKUP_FILE \
  --exclude="${JENKINS_HOME}/workspace" \
  --exclude="${JENKINS_HOME}/caches" \
  --exclude="${JENKINS_HOME}/logs" \
  --exclude="${JENKINS_HOME}/.git" \
  $JENKINS_HOME

# Restart Jenkins
# systemctl start jenkins

# Upload to S3
aws s3 cp $BACKUP_FILE s3://my-jenkins-backups/

# Keep last 30 days
find $BACKUP_DIR -name "jenkins_*.tar.gz" -mtime +30 -delete

echo "Backup completed: $BACKUP_FILE"

# ===== THIN BACKUP PLUGIN =====
# Manages incremental backups within Jenkins UI
# Configure: Jenkins → Manage → ThinBackup
# - Backup directory
# - Schedule: H 1 * * *  (daily at 1am)
# - Max number of backups: 30
# - Wait for idle: true

# ===== JENKINS ON KUBERNETES (HA) =====
# Use Kubernetes deployment for auto-restart
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: jenkins
  namespace: jenkins
spec:
  replicas: 1           # Jenkins is NOT truly HA (active/passive only)
  serviceName: jenkins
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccountName: jenkins
      containers:
        - name: jenkins
          image: jenkins/jenkins:lts-jdk17
          ports:
            - containerPort: 8080
            - containerPort: 50000   # Agent JNLP port
          env:
            - name: JAVA_OPTS
              value: "-Djenkins.install.runSetupWizard=false -Xmx2g -Xms1g"
            - name: CASC_JENKINS_CONFIG
              value: /var/jenkins_home/casc_configs
          volumeMounts:
            - name: jenkins-home
              mountPath: /var/jenkins_home
            - name: casc-config
              mountPath: /var/jenkins_home/casc_configs
          resources:
            requests: { cpu: 500m, memory: 2Gi }
            limits: { cpu: 2, memory: 4Gi }
          livenessProbe:
            httpGet: { path: /login, port: 8080 }
            initialDelaySeconds: 90
            periodSeconds: 10
          readinessProbe:
            httpGet: { path: /login, port: 8080 }
            initialDelaySeconds: 60
      volumes:
        - name: casc-config
          configMap:
            name: jenkins-casc
  volumeClaimTemplates:
    - metadata:
        name: jenkins-home
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 50Gi
```

---

## Essential Plugins

### 🟡 Q17. What are essential Jenkins plugins?

```
PIPELINE:
  Pipeline                      — Core pipeline support
  Pipeline: GitHub Groovy Libraries — Shared libraries from GitHub
  Blue Ocean                    — Modern UI for pipelines
  Pipeline Graph View           — Visualize pipeline stages

SCM:
  Git                           — Git integration
  GitHub Branch Source          — Multibranch from GitHub
  GitLab Branch Source          — Multibranch from GitLab
  GitHub                        — GitHub webhooks + status

AGENTS:
  Kubernetes                    — Dynamic K8s pods as agents
  Amazon EC2                    — EC2 spot instances as agents
  Docker                        — Docker containers as agents
  SSH Build Agents              — SSH-connected agents

CREDENTIALS:
  Credentials Binding           — Use credentials in pipelines
  HashiCorp Vault               — Vault credentials provider
  AWS Credentials               — AWS access keys

TESTING:
  JUnit                         — Publish test results
  HTML Publisher                — Publish HTML reports
  Jacoco                        — Java code coverage
  Cobertura                     — Code coverage
  Performance                   — JMeter/Gatling results
  Warnings Next Generation      — Static analysis results

CODE QUALITY:
  SonarQube Scanner             — SonarQube integration
  Checkstyle                    — Java style
  PMD                           — Java static analysis

NOTIFICATIONS:
  Slack Notification            — Slack integration
  Email Extension               — Rich email notifications
  PagerDuty                     — PagerDuty alerts

CONFIG AS CODE:
  Configuration as Code (JCasC) — YAML-based Jenkins config

SECURITY:
  Role-based Authorization Strategy — RBAC
  LDAP                          — LDAP authentication
  SAML                          — SSO/SAML

UTILITIES:
  Timestamper                   — Add timestamps to log
  AnsiColor                     — Colored console output
  Build Timeout                 — Auto-timeout builds
  Rebuilder                     — Rebuild with same params
  Thin Backup                   — Jenkins backup
  Workspace Cleanup             — Clean workspace
  Job DSL                       — Create jobs as code
  ThinBackup                    — Backup Jenkins config
```

---

## Master Cheatsheet

### Jenkins Pipeline Syntax
```groovy
agent any / none / { label 'linux' } / { docker { image 'node:20' } } / { kubernetes { yaml "..." } }
options { timeout(30,'MINUTES') retry(2) timestamps() disableConcurrentBuilds() }
triggers { cron('H 2 * * *') githubPush() pollSCM('H/5 * * * *') }
when { branch 'main' } { tag 'v*' } { environment name:'X',value:'Y' } { changeRequest() }
parallel { stage('A'){...} stage('B'){...} }
input { message 'Deploy?' ok 'Yes' }
sh 'command'
sh(script: 'cmd', returnStdout: true).trim()
env.MY_VAR = 'value'
credentials('id') / withCredentials([...]) { ... }
stash name:'x', includes:'dist/**' / unstash 'x'
archiveArtifacts 'dist/**'
junit '**/test-results/*.xml'
publishHTML([reportDir:'coverage', reportFiles:'index.html', reportName:'Coverage'])
slackSend channel:'#ci', color:'good', message:'OK'
```

### Useful Groovy Snippets
```groovy
// Get git info
env.GIT_COMMIT_SHORT = sh(script:'git rev-parse --short HEAD', returnStdout:true).trim()
env.GIT_BRANCH_CLEAN = env.BRANCH_NAME.replaceAll('/', '-')

// Read file
def config = readJSON file: 'config.json'
def version = readFile('VERSION').trim()

// Write file
writeFile file: 'version.txt', text: "1.0.${env.BUILD_NUMBER}"

// Check if file exists
if (fileExists('package.json')) { sh 'npm install' }

// Run in directory
dir('frontend') { sh 'npm build' }

// Catch errors without failing build
catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
    sh 'npm test'
}

// Retry with sleep
retry(3) {
    sleep 10
    sh 'kubectl get pods'
}
```

### Question Coverage Index
| Q# | Topic | Difficulty |
|---|---|---|
| Q1 | Jenkins introduction & install | 🟢 |
| Q2 | Freestyle vs Pipeline | 🟢 |
| Q3 | Declarative vs Scripted | 🟢 |
| Q4 | Full Declarative Pipeline structure | 🟡 |
| Q5 | when conditions | 🟡 |
| Q6 | Parallel stages & matrix builds | 🟡 |
| Q7 | Shared Libraries | 🔴 |
| Q8 | Multibranch Pipelines | 🟡 |
| Q9 | Webhook triggers | 🟡 |
| Q10 | Agents: Docker, K8s, EC2 | 🟡 |
| Q11 | Credentials management | 🟡 |
| Q12 | Security best practices | 🔴 |
| Q13 | Artifacts, JUnit, Coverage | 🟡 |
| Q14 | Slack & email notifications | 🟡 |
| Q15 | Jenkins Configuration as Code | 🔴 |
| Q16 | HA & Backup | 🔴 |
| Q17 | Essential plugins | 🟡 |
