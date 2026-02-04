# Jenkins Shared Libraries

## What are Shared Libraries?

Shared Libraries allow you to share common pipeline code across multiple projects.

```
┌─────────────────────────────────────────────────────────────────┐
│                 Shared Library Concept                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Without Shared Library          With Shared Library           │
│   ┌─────────────────────┐        ┌─────────────────────┐       │
│   │ Pipeline A          │        │ Pipeline A          │       │
│   │   def deploy() {..} │        │   myLib.deploy()    │       │
│   │   def test() {..}   │        │   myLib.test()      │       │
│   └─────────────────────┘        └─────────────────────┘       │
│   ┌─────────────────────┐                  │                   │
│   │ Pipeline B          │                  │                   │
│   │   def deploy() {..} │        ┌─────────▼─────────┐         │
│   │   def test() {..}   │        │  Shared Library   │         │
│   └─────────────────────┘        │  • deploy()       │         │
│   ┌─────────────────────┐        │  • test()         │         │
│   │ Pipeline C          │        │  • notify()       │         │
│   │   def deploy() {..} │        └─────────▲─────────┘         │
│   │   def test() {..}   │                  │                   │
│   └─────────────────────┘        ┌─────────┴─────────┐         │
│                                  │ Pipeline B, C...  │         │
│   Duplicated code!               └───────────────────┘         │
│                                  Reusable code!                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Library Structure

```
shared-library/
├── vars/                    # Global variables/functions
│   ├── myPipeline.groovy   # Called as myPipeline()
│   ├── myPipeline.txt      # Help text (optional)
│   ├── log.groovy          # Called as log.info(), log.error()
│   ├── buildApp.groovy
│   └── deployApp.groovy
├── src/                     # Groovy source files
│   └── org/
│       └── company/
│           ├── Utils.groovy
│           └── Docker.groovy
├── resources/               # Non-Groovy files
│   └── templates/
│       └── email.html
└── README.md
```

## Creating a Shared Library

### Global Variable (vars/)

```groovy
// vars/sayHello.groovy
def call(String name = 'World') {
    echo "Hello, ${name}!"
}
```

```groovy
// Usage in Jenkinsfile
@Library('my-shared-library') _

pipeline {
    agent any
    stages {
        stage('Greet') {
            steps {
                sayHello 'Jenkins'
            }
        }
    }
}
```

### Function with Multiple Methods

```groovy
// vars/log.groovy
def info(String message) {
    echo "[INFO] ${message}"
}

def warning(String message) {
    echo "[WARNING] ${message}"
}

def error(String message) {
    echo "[ERROR] ${message}"
}
```

```groovy
// Usage
log.info 'Starting build'
log.warning 'Deprecation notice'
log.error 'Build failed'
```

### Custom Pipeline Step

```groovy
// vars/buildMaven.groovy
def call(Map config = [:]) {
    def mavenVersion = config.mavenVersion ?: '3.9'
    def javaVersion = config.javaVersion ?: '17'
    def goals = config.goals ?: 'clean package'

    stage('Maven Build') {
        docker.image("maven:${mavenVersion}-eclipse-temurin-${javaVersion}").inside {
            sh "mvn ${goals}"
        }
    }
}
```

```groovy
// Usage
buildMaven(
    mavenVersion: '3.9',
    javaVersion: '17',
    goals: 'clean package -DskipTests'
)
```

### Complete Pipeline Template

```groovy
// vars/standardPipeline.groovy
def call(Map config) {
    pipeline {
        agent any

        environment {
            APP_NAME = config.appName
            DEPLOY_ENV = config.deployEnv ?: 'dev'
        }

        stages {
            stage('Checkout') {
                steps {
                    checkout scm
                }
            }

            stage('Build') {
                steps {
                    sh config.buildCommand ?: 'mvn clean package'
                }
            }

            stage('Test') {
                when {
                    expression { config.runTests != false }
                }
                steps {
                    sh config.testCommand ?: 'mvn test'
                }
            }

            stage('Deploy') {
                steps {
                    sh "deploy.sh ${env.APP_NAME} ${env.DEPLOY_ENV}"
                }
            }
        }

        post {
            always {
                cleanWs()
            }
            failure {
                script {
                    if (config.notifyOnFailure) {
                        emailext(
                            subject: "Build Failed: ${env.APP_NAME}",
                            body: "Check: ${env.BUILD_URL}",
                            to: config.notifyEmail
                        )
                    }
                }
            }
        }
    }
}
```

```groovy
// Usage in Jenkinsfile
@Library('my-shared-library') _

standardPipeline(
    appName: 'my-app',
    deployEnv: 'staging',
    runTests: true,
    notifyOnFailure: true,
    notifyEmail: 'team@example.com'
)
```

## Source Classes (src/)

### Creating a Utility Class

```groovy
// src/org/company/Utils.groovy
package org.company

class Utils implements Serializable {
    def script

    Utils(script) {
        this.script = script
    }

    def runShell(String command) {
        script.sh(script: command, returnStdout: true).trim()
    }

    def getGitCommit() {
        return runShell('git rev-parse --short HEAD')
    }

    def getGitBranch() {
        return runShell('git rev-parse --abbrev-ref HEAD')
    }

    static String sanitizeName(String name) {
        return name.toLowerCase().replaceAll(/[^a-z0-9]/, '-')
    }
}
```

```groovy
// vars/utils.groovy
import org.company.Utils

def call() {
    return new Utils(this)
}
```

```groovy
// Usage
def u = utils()
echo "Commit: ${u.getGitCommit()}"
echo "Branch: ${u.getGitBranch()}"
```

### Docker Utility Class

```groovy
// src/org/company/Docker.groovy
package org.company

class Docker implements Serializable {
    def script
    String registry
    String credentialsId

    Docker(script, String registry, String credentialsId) {
        this.script = script
        this.registry = registry
        this.credentialsId = credentialsId
    }

    def build(String imageName, String tag = 'latest') {
        script.sh "docker build -t ${registry}/${imageName}:${tag} ."
    }

    def push(String imageName, String tag = 'latest') {
        script.withCredentials([
            script.usernamePassword(
                credentialsId: credentialsId,
                usernameVariable: 'DOCKER_USER',
                passwordVariable: 'DOCKER_PASS'
            )
        ]) {
            script.sh """
                echo \$DOCKER_PASS | docker login ${registry} -u \$DOCKER_USER --password-stdin
                docker push ${registry}/${imageName}:${tag}
            """
        }
    }

    def buildAndPush(String imageName, String tag = 'latest') {
        build(imageName, tag)
        push(imageName, tag)
    }
}
```

```groovy
// vars/dockerUtil.groovy
import org.company.Docker

def call(Map config) {
    return new Docker(
        this,
        config.registry ?: 'docker.io',
        config.credentialsId ?: 'docker-hub'
    )
}
```

```groovy
// Usage
def docker = dockerUtil(
    registry: 'registry.example.com',
    credentialsId: 'docker-creds'
)
docker.buildAndPush('myapp', "${BUILD_NUMBER}")
```

## Resources

### Using Resource Files

```groovy
// vars/sendEmail.groovy
def call(Map config) {
    // Load template from resources
    def template = libraryResource('templates/email.html')

    // Replace placeholders
    def body = template
        .replace('{{JOB_NAME}}', env.JOB_NAME)
        .replace('{{BUILD_NUMBER}}', env.BUILD_NUMBER)
        .replace('{{STATUS}}', config.status)

    emailext(
        subject: "${config.status}: ${env.JOB_NAME}",
        body: body,
        mimeType: 'text/html',
        to: config.recipients
    )
}
```

```html
<!-- resources/templates/email.html -->
<!DOCTYPE html>
<html>
<body>
    <h2>Build {{STATUS}}</h2>
    <p>Job: {{JOB_NAME}}</p>
    <p>Build: #{{BUILD_NUMBER}}</p>
</body>
</html>
```

## Configuring Shared Libraries

### Global Configuration

```
Manage Jenkins → System → Global Pipeline Libraries

Name: my-shared-library
Default version: main
Allow default version to be overridden: ✓
Include @Library changes in job recent changes: ✓

Retrieval method: Modern SCM
  Source Code Management: Git
  Project Repository: https://github.com/company/jenkins-shared-library.git
  Credentials: github-token
```

### Folder-Level Library

```
Folder → Configure → Pipeline Libraries

Name: team-library
Default version: main
Source Code Management: Git
Repository: https://github.com/company/team-library.git
```

## Using Libraries in Pipelines

### Import Methods

```groovy
// Import entire library
@Library('my-shared-library') _

// Import specific version
@Library('my-shared-library@v1.0.0') _

// Import from branch
@Library('my-shared-library@feature-branch') _

// Import multiple libraries
@Library(['my-shared-library', 'other-library']) _

// Import with underscore to load implicitly
@Library('my-shared-library') _

// Import and assign to variable
@Library('my-shared-library') import org.company.Utils
```

### Dynamic Library Loading

```groovy
pipeline {
    agent any
    stages {
        stage('Load Library') {
            steps {
                script {
                    library 'my-shared-library@main'
                }
            }
        }
        stage('Use Library') {
            steps {
                sayHello 'World'
            }
        }
    }
}
```

## Complete Library Example

### vars/ciPipeline.groovy

```groovy
def call(Map config) {
    def appName = config.appName ?: error('appName is required')
    def buildTool = config.buildTool ?: 'maven'

    pipeline {
        agent any

        options {
            timeout(time: 1, unit: 'HOURS')
            buildDiscarder(logRotator(numToKeepStr: '10'))
            timestamps()
        }

        environment {
            APP_NAME = appName
            VERSION = "${BUILD_NUMBER}"
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
                    script {
                        switch(buildTool) {
                            case 'maven':
                                sh 'mvn clean package -DskipTests'
                                break
                            case 'gradle':
                                sh './gradlew build -x test'
                                break
                            case 'npm':
                                sh 'npm ci && npm run build'
                                break
                            default:
                                error "Unknown build tool: ${buildTool}"
                        }
                    }
                }
            }

            stage('Test') {
                when {
                    expression { config.skipTests != true }
                }
                steps {
                    script {
                        switch(buildTool) {
                            case 'maven':
                                sh 'mvn test'
                                junit 'target/surefire-reports/*.xml'
                                break
                            case 'gradle':
                                sh './gradlew test'
                                junit 'build/test-results/test/*.xml'
                                break
                            case 'npm':
                                sh 'npm test'
                                break
                        }
                    }
                }
            }

            stage('Docker Build') {
                when {
                    expression { config.dockerBuild == true }
                }
                steps {
                    sh """
                        docker build -t ${config.dockerRegistry}/${APP_NAME}:${GIT_COMMIT_SHORT} .
                        docker tag ${config.dockerRegistry}/${APP_NAME}:${GIT_COMMIT_SHORT} \
                            ${config.dockerRegistry}/${APP_NAME}:latest
                    """
                }
            }
        }

        post {
            always {
                cleanWs()
            }
            failure {
                script {
                    if (config.slackChannel) {
                        slackSend(
                            channel: config.slackChannel,
                            color: 'danger',
                            message: "Build Failed: ${APP_NAME} #${BUILD_NUMBER}"
                        )
                    }
                }
            }
        }
    }
}
```

### Usage

```groovy
// Jenkinsfile
@Library('my-shared-library') _

ciPipeline(
    appName: 'my-service',
    buildTool: 'maven',
    skipTests: false,
    dockerBuild: true,
    dockerRegistry: 'registry.example.com',
    slackChannel: '#builds'
)
```

## Quick Reference

### Library Structure

| Directory | Purpose |
|-----------|---------|
| `vars/` | Global variables and functions |
| `src/` | Groovy classes |
| `resources/` | Non-Groovy files |

### Import Syntax

| Syntax | Description |
|--------|-------------|
| `@Library('name') _` | Import library |
| `@Library('name@version') _` | Specific version |
| `@Library(['lib1', 'lib2']) _` | Multiple libraries |
| `library 'name'` | Dynamic import |

---

**Previous:** [08-jenkins-pipeline-advanced.md](08-jenkins-pipeline-advanced.md) | **Next:** [10-jenkins-build-agents.md](10-jenkins-build-agents.md)
