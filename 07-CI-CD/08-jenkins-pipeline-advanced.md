# Jenkins Pipeline Advanced

## Parallel Stages

### Basic Parallel Execution

```groovy
pipeline {
    agent any

    stages {
        stage('Parallel Tests') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'mvn test -Dtest=*UnitTest'
                    }
                }
                stage('Integration Tests') {
                    steps {
                        sh 'mvn test -Dtest=*IntegrationTest'
                    }
                }
                stage('E2E Tests') {
                    steps {
                        sh 'npm run test:e2e'
                    }
                }
            }
        }
    }
}
```

### Parallel with Different Agents

```groovy
pipeline {
    agent none

    stages {
        stage('Build and Test') {
            parallel {
                stage('Linux') {
                    agent { label 'linux' }
                    steps {
                        sh 'make build'
                        sh 'make test'
                    }
                }
                stage('Windows') {
                    agent { label 'windows' }
                    steps {
                        bat 'msbuild build.proj'
                        bat 'vstest.console.exe tests.dll'
                    }
                }
                stage('macOS') {
                    agent { label 'macos' }
                    steps {
                        sh 'xcodebuild build'
                        sh 'xcodebuild test'
                    }
                }
            }
        }
    }
}
```

### Fail Fast

```groovy
stage('Parallel Tests') {
    failFast true  // Stop all parallel stages if one fails
    parallel {
        stage('Test A') {
            steps { sh 'run-tests-a.sh' }
        }
        stage('Test B') {
            steps { sh 'run-tests-b.sh' }
        }
    }
}
```

## Conditional Execution (when)

### Branch Conditions

```groovy
stages {
    stage('Deploy to Dev') {
        when {
            branch 'develop'
        }
        steps {
            sh './deploy.sh dev'
        }
    }

    stage('Deploy to Staging') {
        when {
            branch 'main'
        }
        steps {
            sh './deploy.sh staging'
        }
    }

    stage('Deploy to Prod') {
        when {
            tag pattern: "v\\d+\\.\\d+\\.\\d+", comparator: "REGEXP"
        }
        steps {
            sh './deploy.sh prod'
        }
    }
}
```

### Environment Conditions

```groovy
stage('Deploy') {
    when {
        environment name: 'DEPLOY_TO', value: 'production'
    }
    steps {
        sh './deploy.sh'
    }
}
```

### Expression Conditions

```groovy
stage('Conditional Stage') {
    when {
        expression {
            return params.RUN_TESTS == true
        }
    }
    steps {
        sh 'mvn test'
    }
}

stage('Deploy Only on Success') {
    when {
        expression {
            return currentBuild.result == null || currentBuild.result == 'SUCCESS'
        }
    }
    steps {
        sh './deploy.sh'
    }
}
```

### Multiple Conditions

```groovy
stage('Production Deploy') {
    when {
        allOf {
            branch 'main'
            environment name: 'DEPLOY_ENABLED', value: 'true'
        }
    }
    steps {
        sh './deploy-prod.sh'
    }
}

stage('Any Condition') {
    when {
        anyOf {
            branch 'main'
            branch 'develop'
            tag pattern: "v.*"
        }
    }
    steps {
        sh './build.sh'
    }
}

stage('Not Condition') {
    when {
        not {
            branch 'main'
        }
    }
    steps {
        echo 'Not on main branch'
    }
}
```

### Before Agent

```groovy
stage('Expensive Stage') {
    when {
        beforeAgent true  // Evaluate before allocating agent
        branch 'main'
    }
    agent {
        label 'expensive-agent'
    }
    steps {
        sh './expensive-operation.sh'
    }
}
```

## Input and Approval

### Basic Input

```groovy
stage('Deploy to Production') {
    steps {
        input message: 'Deploy to production?', ok: 'Deploy'
        sh './deploy-prod.sh'
    }
}
```

### Input with Parameters

```groovy
stage('Approval') {
    steps {
        script {
            def userInput = input(
                id: 'userInput',
                message: 'Promote to production?',
                parameters: [
                    choice(
                        name: 'TARGET_ENV',
                        choices: ['staging', 'production'],
                        description: 'Select environment'
                    ),
                    booleanParam(
                        name: 'NOTIFY_USERS',
                        defaultValue: true,
                        description: 'Send notifications?'
                    ),
                    string(
                        name: 'VERSION',
                        defaultValue: '1.0.0',
                        description: 'Version to deploy'
                    )
                ]
            )
            env.TARGET_ENV = userInput.TARGET_ENV
            env.NOTIFY_USERS = userInput.NOTIFY_USERS
            env.VERSION = userInput.VERSION
        }
        sh "echo Deploying ${VERSION} to ${TARGET_ENV}"
    }
}
```

### Input with Timeout

```groovy
stage('Manual Approval') {
    steps {
        timeout(time: 1, unit: 'HOURS') {
            input message: 'Approve deployment?',
                  submitter: 'admin,release-managers'
        }
        sh './deploy.sh'
    }
}
```

### Input at Stage Level

```groovy
stage('Production') {
    input {
        message "Deploy to production?"
        ok "Deploy"
        submitter "admin,devops"
        parameters {
            string(name: 'VERSION', defaultValue: '', description: 'Version')
        }
    }
    steps {
        sh "./deploy.sh ${VERSION}"
    }
}
```

## Script Block

### Using Groovy in Declarative Pipeline

```groovy
stage('Complex Logic') {
    steps {
        script {
            // Groovy code
            def servers = ['server1', 'server2', 'server3']

            for (server in servers) {
                echo "Deploying to ${server}"
                sh "ssh ${server} 'deploy.sh'"
            }

            // Conditional logic
            if (env.BRANCH_NAME == 'main') {
                sh 'notify-release.sh'
            }

            // Try-catch
            try {
                sh 'risky-command.sh'
            } catch (Exception e) {
                echo "Command failed: ${e.message}"
                currentBuild.result = 'UNSTABLE'
            }
        }
    }
}
```

### Defining Functions

```groovy
def notifySlack(String status) {
    def color = status == 'SUCCESS' ? 'good' : 'danger'
    slackSend(
        channel: '#builds',
        color: color,
        message: "${env.JOB_NAME} #${env.BUILD_NUMBER} - ${status}"
    )
}

def deployToEnvironment(String env) {
    sh """
        kubectl config use-context ${env}
        kubectl apply -f k8s/
    """
}

pipeline {
    agent any

    stages {
        stage('Deploy') {
            steps {
                script {
                    deployToEnvironment('staging')
                }
            }
        }
    }

    post {
        success {
            script {
                notifySlack('SUCCESS')
            }
        }
        failure {
            script {
                notifySlack('FAILURE')
            }
        }
    }
}
```

## Matrix Builds

```groovy
pipeline {
    agent any

    stages {
        stage('Build Matrix') {
            matrix {
                axes {
                    axis {
                        name 'PLATFORM'
                        values 'linux', 'windows', 'mac'
                    }
                    axis {
                        name 'JAVA_VERSION'
                        values '11', '17', '21'
                    }
                }
                excludes {
                    exclude {
                        axis {
                            name 'PLATFORM'
                            values 'mac'
                        }
                        axis {
                            name 'JAVA_VERSION'
                            values '11'
                        }
                    }
                }
                stages {
                    stage('Build') {
                        steps {
                            echo "Building on ${PLATFORM} with Java ${JAVA_VERSION}"
                            sh "./build.sh ${PLATFORM} ${JAVA_VERSION}"
                        }
                    }
                    stage('Test') {
                        steps {
                            sh "./test.sh ${PLATFORM} ${JAVA_VERSION}"
                        }
                    }
                }
            }
        }
    }
}
```

## Stash and Unstash

### Share Files Between Stages

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
                stash includes: 'target/*.jar', name: 'app-jar'
            }
        }

        stage('Test on Multiple Nodes') {
            parallel {
                stage('Test Linux') {
                    agent { label 'linux' }
                    steps {
                        unstash 'app-jar'
                        sh 'java -jar target/app.jar --test'
                    }
                }
                stage('Test Windows') {
                    agent { label 'windows' }
                    steps {
                        unstash 'app-jar'
                        bat 'java -jar target\\app.jar --test'
                    }
                }
            }
        }
    }
}
```

## Error Handling

### Try-Catch-Finally

```groovy
stage('Risky Stage') {
    steps {
        script {
            try {
                sh 'might-fail.sh'
            } catch (Exception e) {
                echo "Caught: ${e}"
                currentBuild.result = 'UNSTABLE'
            } finally {
                sh 'cleanup.sh'
            }
        }
    }
}
```

### catchError

```groovy
stage('Test') {
    steps {
        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
            sh 'run-tests.sh'
        }
    }
}
```

### warnError

```groovy
stage('Optional Step') {
    steps {
        warnError('Script failed but continuing') {
            sh 'optional-script.sh'
        }
    }
}
```

## Milestone and Lock

### Milestone

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                milestone(1)
                sh 'mvn clean package'
            }
        }
        stage('Deploy') {
            steps {
                milestone(2)  // Cancels older builds at this point
                sh './deploy.sh'
            }
        }
    }
}
```

### Lock Resources

```groovy
stage('Deploy') {
    steps {
        lock(resource: 'production-server') {
            sh './deploy.sh'
        }
    }
}

// With timeout
stage('Deploy') {
    steps {
        lock(resource: 'staging-server', inversePrecedence: true) {
            timeout(time: 10, unit: 'MINUTES') {
                sh './deploy.sh'
            }
        }
    }
}
```

## Complete Advanced Pipeline

```groovy
pipeline {
    agent none

    options {
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }

    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'])
        booleanParam(name: 'SKIP_TESTS', defaultValue: false)
    }

    stages {
        stage('Build') {
            agent { docker 'maven:3.9' }
            steps {
                sh 'mvn clean package -DskipTests'
                stash includes: 'target/*.jar', name: 'jar'
            }
        }

        stage('Test') {
            when {
                expression { !params.SKIP_TESTS }
            }
            parallel {
                stage('Unit Tests') {
                    agent { docker 'maven:3.9' }
                    steps {
                        unstash 'jar'
                        sh 'mvn test'
                    }
                    post {
                        always {
                            junit 'target/surefire-reports/*.xml'
                        }
                    }
                }
                stage('Integration Tests') {
                    agent { docker 'maven:3.9' }
                    steps {
                        unstash 'jar'
                        sh 'mvn verify -Pintegration'
                    }
                }
            }
        }

        stage('Deploy to Dev') {
            when { expression { params.ENVIRONMENT == 'dev' } }
            agent { label 'deploy' }
            steps {
                unstash 'jar'
                sh './deploy.sh dev'
            }
        }

        stage('Deploy to Staging') {
            when { expression { params.ENVIRONMENT == 'staging' } }
            agent { label 'deploy' }
            steps {
                unstash 'jar'
                sh './deploy.sh staging'
            }
        }

        stage('Deploy to Production') {
            when { expression { params.ENVIRONMENT == 'prod' } }
            agent { label 'deploy' }
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: 'Deploy to production?',
                          submitter: 'admin,release-team'
                }
                lock(resource: 'production') {
                    unstash 'jar'
                    sh './deploy.sh prod'
                }
            }
        }
    }

    post {
        success { echo 'Pipeline succeeded!' }
        failure { echo 'Pipeline failed!' }
    }
}
```

## Quick Reference

### When Conditions

| Condition | Description |
|-----------|-------------|
| `branch` | Match branch name |
| `tag` | Match tag |
| `environment` | Check env variable |
| `expression` | Groovy expression |
| `allOf` | All conditions |
| `anyOf` | Any condition |
| `not` | Negate condition |

### Stage Options

| Option | Description |
|--------|-------------|
| `failFast` | Stop parallel on failure |
| `input` | Wait for approval |
| `when` | Conditional execution |
| `options` | Stage-specific options |

---

**Previous:** [07-jenkins-pipelines.md](07-jenkins-pipelines.md) | **Next:** [09-jenkins-shared-libraries.md](09-jenkins-shared-libraries.md)
