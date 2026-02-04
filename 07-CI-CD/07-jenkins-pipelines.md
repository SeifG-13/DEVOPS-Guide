# Jenkins Pipelines

## What is a Jenkins Pipeline?

Pipeline is a suite of plugins that supports implementing continuous delivery pipelines as code.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Pipeline Concepts                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Jenkinsfile (Pipeline as Code)                                │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  pipeline {                                              │   │
│   │      agent any                                           │   │
│   │      stages {                                            │   │
│   │          stage('Build') { steps { ... } }               │   │
│   │          stage('Test')  { steps { ... } }               │   │
│   │          stage('Deploy'){ steps { ... } }               │   │
│   │      }                                                   │   │
│   │  }                                                       │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Benefits:                                                     │
│   • Version controlled                                          │
│   • Code review for pipeline changes                            │
│   • Audit trail                                                 │
│   • Single source of truth                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Declarative vs Scripted

| Feature | Declarative | Scripted |
|---------|-------------|----------|
| Syntax | Structured, opinionated | Flexible, Groovy-based |
| Learning Curve | Easier | Steeper |
| Error Handling | Built-in | Manual |
| Validation | Syntax validation | Runtime errors |
| Recommendation | Preferred for most cases | Complex logic |

### Declarative Pipeline

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                echo 'Building...'
                sh 'mvn clean compile'
            }
        }
        stage('Test') {
            steps {
                echo 'Testing...'
                sh 'mvn test'
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying...'
                sh 'mvn deploy'
            }
        }
    }
}
```

### Scripted Pipeline

```groovy
node {
    stage('Build') {
        echo 'Building...'
        sh 'mvn clean compile'
    }
    stage('Test') {
        echo 'Testing...'
        sh 'mvn test'
    }
    stage('Deploy') {
        echo 'Deploying...'
        sh 'mvn deploy'
    }
}
```

## Pipeline Structure

```groovy
pipeline {
    // Where to run
    agent any

    // Pipeline options
    options {
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    // Environment variables
    environment {
        APP_NAME = 'myapp'
        VERSION = '1.0.0'
    }

    // Build parameters
    parameters {
        string(name: 'BRANCH', defaultValue: 'main')
        booleanParam(name: 'RUN_TESTS', defaultValue: true)
    }

    // Build triggers
    triggers {
        pollSCM('H/5 * * * *')
    }

    // Pipeline stages
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }

    // Post-build actions
    post {
        always {
            cleanWs()
        }
        success {
            echo 'Build succeeded!'
        }
        failure {
            echo 'Build failed!'
        }
    }
}
```

## Agent Directive

### Agent Options

```groovy
// Run on any available agent
agent any

// Run on controller (not recommended)
agent none

// Run on specific label
agent {
    label 'linux'
}

// Run in Docker container
agent {
    docker {
        image 'maven:3.9-eclipse-temurin-17'
        args '-v $HOME/.m2:/root/.m2'
    }
}

// Run on Kubernetes
agent {
    kubernetes {
        yaml '''
        apiVersion: v1
        kind: Pod
        spec:
          containers:
          - name: maven
            image: maven:3.9-eclipse-temurin-17
            command:
            - cat
            tty: true
        '''
    }
}
```

### Per-Stage Agent

```groovy
pipeline {
    agent none

    stages {
        stage('Build') {
            agent {
                docker { image 'maven:3.9' }
            }
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Test') {
            agent {
                docker { image 'node:20' }
            }
            steps {
                sh 'npm test'
            }
        }
    }
}
```

## Stages and Steps

### Basic Stages

```groovy
stages {
    stage('Checkout') {
        steps {
            checkout scm
        }
    }
    stage('Build') {
        steps {
            sh 'mvn clean compile'
        }
    }
    stage('Test') {
        steps {
            sh 'mvn test'
        }
    }
    stage('Package') {
        steps {
            sh 'mvn package -DskipTests'
        }
    }
    stage('Deploy') {
        steps {
            sh './deploy.sh'
        }
    }
}
```

### Common Steps

```groovy
steps {
    // Shell command
    sh 'echo "Hello"'
    sh '''
        echo "Multi-line"
        echo "Script"
    '''

    // Windows batch
    bat 'echo Hello'

    // PowerShell
    powershell 'Write-Host "Hello"'

    // Print message
    echo 'Hello, World!'

    // Change directory
    dir('subdirectory') {
        sh 'pwd'
    }

    // Set environment
    withEnv(['VAR=value']) {
        sh 'echo $VAR'
    }

    // Sleep
    sleep(time: 10, unit: 'SECONDS')

    // Retry
    retry(3) {
        sh 'flaky-command.sh'
    }

    // Timeout
    timeout(time: 5, unit: 'MINUTES') {
        sh 'long-running-command.sh'
    }
}
```

## Environment Variables

### Defining Environment Variables

```groovy
pipeline {
    agent any

    environment {
        // Global environment
        APP_NAME = 'myapp'
        VERSION = "${env.BUILD_NUMBER}"

        // From credentials
        AWS_CREDS = credentials('aws-credentials')
        API_KEY = credentials('api-key')
    }

    stages {
        stage('Build') {
            environment {
                // Stage-specific environment
                STAGE_VAR = 'build'
            }
            steps {
                sh 'echo "Building $APP_NAME version $VERSION"'
            }
        }
    }
}
```

### Built-in Environment Variables

```groovy
steps {
    sh '''
        echo "Build Number: ${BUILD_NUMBER}"
        echo "Build ID: ${BUILD_ID}"
        echo "Build URL: ${BUILD_URL}"
        echo "Job Name: ${JOB_NAME}"
        echo "Workspace: ${WORKSPACE}"
        echo "Jenkins Home: ${JENKINS_HOME}"
        echo "Node Name: ${NODE_NAME}"
    '''
}
```

## Parameters

```groovy
pipeline {
    agent any

    parameters {
        string(
            name: 'BRANCH_NAME',
            defaultValue: 'main',
            description: 'Branch to build'
        )
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'staging', 'prod'],
            description: 'Deployment environment'
        )
        booleanParam(
            name: 'RUN_TESTS',
            defaultValue: true,
            description: 'Run tests?'
        )
        password(
            name: 'SECRET',
            defaultValue: '',
            description: 'Secret value'
        )
        text(
            name: 'CONFIG',
            defaultValue: '',
            description: 'Configuration text'
        )
    }

    stages {
        stage('Build') {
            steps {
                echo "Branch: ${params.BRANCH_NAME}"
                echo "Environment: ${params.ENVIRONMENT}"

                script {
                    if (params.RUN_TESTS) {
                        sh 'mvn test'
                    }
                }
            }
        }
    }
}
```

## Post Section

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }

    post {
        always {
            // Always runs
            echo 'Pipeline completed'
            cleanWs()
        }
        success {
            // Only on success
            echo 'Build succeeded!'
            archiveArtifacts artifacts: 'target/*.jar'
        }
        failure {
            // Only on failure
            echo 'Build failed!'
            mail to: 'team@example.com',
                 subject: "Failed: ${env.JOB_NAME}",
                 body: "Build ${env.BUILD_NUMBER} failed"
        }
        unstable {
            // When tests fail
            echo 'Build unstable'
        }
        changed {
            // When status changes
            echo 'Build status changed'
        }
        aborted {
            // When build is aborted
            echo 'Build was aborted'
        }
    }
}
```

## Options

```groovy
pipeline {
    agent any

    options {
        // Build timeout
        timeout(time: 1, unit: 'HOURS')

        // Retry entire pipeline
        retry(3)

        // Discard old builds
        buildDiscarder(logRotator(
            numToKeepStr: '10',
            daysToKeepStr: '30',
            artifactNumToKeepStr: '5'
        ))

        // Skip default checkout
        skipDefaultCheckout()

        // Timestamps in console
        timestamps()

        // Disable concurrent builds
        disableConcurrentBuilds()

        // ANSI colors
        ansiColor('xterm')

        // Set build name
        buildName("${BUILD_NUMBER}-${params.ENVIRONMENT}")
    }

    stages {
        stage('Build') {
            steps {
                checkout scm
                sh 'mvn clean package'
            }
        }
    }
}
```

## Triggers

```groovy
pipeline {
    agent any

    triggers {
        // Poll SCM every 5 minutes
        pollSCM('H/5 * * * *')

        // Build periodically (cron)
        cron('H 2 * * *')

        // Upstream job trigger
        upstream(
            upstreamProjects: 'upstream-job',
            threshold: hudson.model.Result.SUCCESS
        )
    }

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }
}
```

## Complete Jenkinsfile Example

```groovy
pipeline {
    agent any

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        APP_NAME = 'myapp'
        DOCKER_REGISTRY = 'registry.example.com'
        DOCKER_IMAGE = "${DOCKER_REGISTRY}/${APP_NAME}"
    }

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'staging', 'prod'],
            description: 'Deployment environment'
        )
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Package') {
            steps {
                sh 'mvn package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar'
            }
        }

        stage('Docker Build') {
            steps {
                sh """
                    docker build -t ${DOCKER_IMAGE}:${GIT_COMMIT_SHORT} .
                    docker tag ${DOCKER_IMAGE}:${GIT_COMMIT_SHORT} ${DOCKER_IMAGE}:latest
                """
            }
        }

        stage('Deploy') {
            when {
                expression { params.ENVIRONMENT != 'prod' }
            }
            steps {
                sh "./deploy.sh ${params.ENVIRONMENT}"
            }
        }

        stage('Deploy to Prod') {
            when {
                expression { params.ENVIRONMENT == 'prod' }
            }
            steps {
                input message: 'Deploy to production?', ok: 'Deploy'
                sh './deploy.sh prod'
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            mail to: 'team@example.com',
                 subject: "Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Check: ${env.BUILD_URL}"
        }
    }
}
```

## Quick Reference

### Pipeline Syntax

| Element | Purpose |
|---------|---------|
| `pipeline` | Root element |
| `agent` | Where to run |
| `stages` | Container for stages |
| `stage` | Named phase |
| `steps` | Actual commands |
| `post` | Post-build actions |
| `environment` | Variables |
| `parameters` | Build parameters |
| `options` | Pipeline options |
| `triggers` | Build triggers |

### Common Steps

| Step | Description |
|------|-------------|
| `sh` | Shell command |
| `bat` | Windows batch |
| `echo` | Print message |
| `checkout scm` | Checkout code |
| `git` | Clone Git repo |
| `archiveArtifacts` | Save artifacts |
| `junit` | Publish test results |

---

**Previous:** [06-jenkins-freestyle-jobs.md](06-jenkins-freestyle-jobs.md) | **Next:** [08-jenkins-pipeline-advanced.md](08-jenkins-pipeline-advanced.md)
