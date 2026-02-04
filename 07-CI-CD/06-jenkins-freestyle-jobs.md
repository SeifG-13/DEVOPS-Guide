# Jenkins Freestyle Jobs

## What is a Freestyle Job?

Freestyle jobs are the traditional Jenkins job type with a web-based configuration.

```
┌─────────────────────────────────────────────────────────────────┐
│                   Freestyle Job Structure                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    General Settings                      │   │
│   │  • Description, parameters, concurrent builds           │   │
│   └─────────────────────────────────────────────────────────┘   │
│                            │                                     │
│                            ▼                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │               Source Code Management                     │   │
│   │  • Git, SVN, or None                                    │   │
│   └─────────────────────────────────────────────────────────┘   │
│                            │                                     │
│                            ▼                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                   Build Triggers                         │   │
│   │  • Poll SCM, webhook, schedule, upstream                │   │
│   └─────────────────────────────────────────────────────────┘   │
│                            │                                     │
│                            ▼                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                 Build Environment                        │   │
│   │  • Credentials, timestamps, cleanup                     │   │
│   └─────────────────────────────────────────────────────────┘   │
│                            │                                     │
│                            ▼                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    Build Steps                           │   │
│   │  • Shell scripts, Maven, Gradle, etc.                   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                            │                                     │
│                            ▼                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                 Post-build Actions                       │   │
│   │  • Archive, email, trigger other jobs                   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Creating a Freestyle Job

### Step 1: Create New Item

```
Dashboard → New Item
  → Enter job name: "my-freestyle-job"
  → Select "Freestyle project"
  → Click OK
```

### Step 2: General Configuration

```
General:
  ✓ Description: Build and deploy my application
  ✓ Discard old builds
      Strategy: Log Rotation
      Days to keep builds: 30
      Max # of builds to keep: 10
  ✓ GitHub project: https://github.com/user/repo
  □ This project is parameterized
  □ Throttle builds
  □ Execute concurrent builds if necessary
```

## Build Parameters

### Adding Parameters

```
General → This project is parameterized → Add Parameter

Parameter Types:
• String Parameter
• Choice Parameter
• Boolean Parameter
• Password Parameter
• File Parameter
• Credentials Parameter
```

### String Parameter

```
Name: BRANCH_NAME
Default Value: main
Description: Git branch to build
```

### Choice Parameter

```
Name: ENVIRONMENT
Choices:
  development
  staging
  production
Description: Deployment environment
```

### Boolean Parameter

```
Name: RUN_TESTS
Default: true
Description: Run unit tests
```

### Using Parameters in Build Steps

```bash
#!/bin/bash

echo "Building branch: $BRANCH_NAME"
echo "Deploying to: $ENVIRONMENT"

if [ "$RUN_TESTS" = "true" ]; then
    mvn test
fi

mvn clean package
```

## Source Code Management

### Git Configuration

```
Source Code Management → Git

Repository URL: https://github.com/user/repo.git
Credentials: Select credentials

Branches to build:
  Branch Specifier: */${BRANCH_NAME}
  # or: */main, */develop, **/feature-*

Repository browser: (Auto)

Additional Behaviours:
  → Clean before checkout
  → Checkout to a sub-directory: src
  → Sparse Checkout paths
```

### Advanced Git Options

```
Additional Behaviours:
  → Clean before checkout
  → Prune stale remote tracking branches
  → Clone timeout: 30 minutes
  → Checkout to specific local branch: ${BRANCH_NAME}
  → Custom user name/e-mail address
  → Polling ignores commits with certain messages
```

## Build Triggers

### Poll SCM

```
Build Triggers → Poll SCM
Schedule: H/5 * * * *  # Every 5 minutes

# Cron syntax:
# MINUTE HOUR DOM MONTH DOW
# H = hash for load distribution
```

### GitHub Hook Trigger

```
Build Triggers → GitHub hook trigger for GITScm polling

# Requires GitHub webhook configured:
# https://jenkins.example.com/github-webhook/
```

### Build Periodically

```
Build Triggers → Build periodically
Schedule: H 2 * * *  # Daily at 2 AM
```

### Trigger from Other Jobs

```
Build Triggers → Build after other projects are built
Projects to watch: upstream-job
Trigger only if build is stable
```

### Remote Trigger

```
Build Triggers → Trigger builds remotely
Authentication Token: my-secret-token

# Trigger URL:
# http://jenkins/job/my-job/build?token=my-secret-token
```

## Build Environment

### Environment Options

```
Build Environment:
  ✓ Delete workspace before build starts
  ✓ Use secret text(s) or file(s)
  ✓ Add timestamps to the Console Output
  ✓ Abort the build if it's stuck
      Timeout: 30 minutes
  ✓ Set Build Name
      #${BUILD_NUMBER} - ${ENVIRONMENT}
```

### Inject Environment Variables

```
Build Environment → Inject environment variables

Properties Content:
APP_NAME=myapp
APP_VERSION=1.0.${BUILD_NUMBER}
JAVA_HOME=/usr/lib/jvm/java-17

# Or from file:
Properties File Path: env.properties
```

## Build Steps

### Execute Shell

```bash
#!/bin/bash
set -e  # Exit on error

echo "=== Build Information ==="
echo "Build Number: ${BUILD_NUMBER}"
echo "Workspace: ${WORKSPACE}"
echo "Job Name: ${JOB_NAME}"

# Install dependencies
npm install

# Run tests
npm test

# Build application
npm run build

# Package
tar -czf app-${BUILD_NUMBER}.tar.gz dist/
```

### Execute Windows Batch Command

```batch
@echo off
echo Building on Windows...

REM Set variables
set BUILD_DIR=%WORKSPACE%\build

REM Clean
if exist %BUILD_DIR% rmdir /s /q %BUILD_DIR%
mkdir %BUILD_DIR%

REM Build
msbuild MyProject.sln /p:Configuration=Release

REM Copy artifacts
copy bin\Release\*.dll %BUILD_DIR%
```

### Invoke Maven

```
Build Steps → Invoke top-level Maven targets

Maven Version: Maven3
Goals: clean package -DskipTests=${SKIP_TESTS}
POM: pom.xml

Advanced:
  JAVA_HOME: /usr/lib/jvm/java-17
  Properties: env=${ENVIRONMENT}
```

### Invoke Gradle

```
Build Steps → Invoke Gradle script

Gradle Version: Gradle7
Tasks: clean build
Build File: build.gradle

Advanced:
  Switches: --stacktrace --info
```

### Multiple Build Steps

```
Build Steps:
  1. Execute shell: Clean and prepare
  2. Invoke Maven: Build
  3. Execute shell: Run integration tests
  4. Execute shell: Package and deploy
```

## Shell Scripting Best Practices

### Error Handling

```bash
#!/bin/bash
set -e          # Exit on error
set -u          # Exit on undefined variable
set -o pipefail # Exit on pipe failure

# Trap errors
trap 'echo "Error on line $LINENO"' ERR

# Function with error handling
build_app() {
    echo "Building application..."
    mvn clean package || {
        echo "Build failed!"
        return 1
    }
}

# Main
build_app
echo "Build successful!"
```

### Logging and Output

```bash
#!/bin/bash

# Colors (if terminal supports)
RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m' # No Color

log_info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1" >&2
}

log_info "Starting build..."
log_info "Build number: ${BUILD_NUMBER}"

if mvn test; then
    log_info "Tests passed!"
else
    log_error "Tests failed!"
    exit 1
fi
```

### Using Jenkins Environment Variables

```bash
#!/bin/bash

# Built-in Jenkins variables
echo "Build Number: ${BUILD_NUMBER}"
echo "Build ID: ${BUILD_ID}"
echo "Build URL: ${BUILD_URL}"
echo "Job Name: ${JOB_NAME}"
echo "Workspace: ${WORKSPACE}"
echo "Jenkins URL: ${JENKINS_URL}"
echo "Node Name: ${NODE_NAME}"
echo "Executor Number: ${EXECUTOR_NUMBER}"

# Git variables (with Git plugin)
echo "Git Branch: ${GIT_BRANCH}"
echo "Git Commit: ${GIT_COMMIT}"
echo "Git URL: ${GIT_URL}"
```

## Post-build Actions

### Archive Artifacts

```
Post-build Actions → Archive the artifacts

Files to archive: target/*.jar, dist/**/*
Excludes: **/*-sources.jar
Fingerprint all archived artifacts: ✓
```

### Publish Test Results

```
Post-build Actions → Publish JUnit test result report

Test report XMLs: target/surefire-reports/*.xml
Health report amplification factor: 1.0
Allow empty results: □
```

### Email Notification

```
Post-build Actions → E-mail Notification

Recipients: team@example.com
Send e-mail for every unstable build: ✓
Send separate e-mails to individuals who broke the build: ✓
```

### Trigger Other Jobs

```
Post-build Actions → Build other projects

Projects to build: deploy-job, notification-job
Trigger only if build is stable: ✓
```

### Workspace Cleanup

```
Post-build Actions → Delete workspace when build is done

Clean when status is: SUCCESS, UNSTABLE, FAILURE, NOT_BUILT, ABORTED
```

## Complete Freestyle Job Example

```
Job: my-application-build

General:
  ✓ Discard old builds (keep 10)
  ✓ This project is parameterized
      String: BRANCH_NAME = main
      Choice: ENVIRONMENT = [dev, staging, prod]
      Boolean: RUN_TESTS = true

Source Code Management:
  Git:
    URL: https://github.com/company/myapp.git
    Credentials: github-token
    Branch: */${BRANCH_NAME}

Build Triggers:
  ✓ GitHub hook trigger for GITScm polling
  ✓ Poll SCM: H/5 * * * *

Build Environment:
  ✓ Delete workspace before build starts
  ✓ Add timestamps to Console Output
  ✓ Use secret text(s) or file(s)
      Secret text: API_KEY

Build Steps:
  Execute shell:
    #!/bin/bash
    set -e

    echo "Building ${JOB_NAME} #${BUILD_NUMBER}"
    echo "Branch: ${BRANCH_NAME}"
    echo "Environment: ${ENVIRONMENT}"

    # Build
    mvn clean package

    # Tests
    if [ "${RUN_TESTS}" = "true" ]; then
        mvn test
    fi

    # Create version file
    echo "${BUILD_NUMBER}" > version.txt

Post-build Actions:
  ✓ Archive artifacts: target/*.jar, version.txt
  ✓ Publish JUnit results: target/surefire-reports/*.xml
  ✓ E-mail notification: team@example.com
  ✓ Build other projects: deploy-${ENVIRONMENT}
```

## Quick Reference

### Common Environment Variables

| Variable | Description |
|----------|-------------|
| `BUILD_NUMBER` | Current build number |
| `BUILD_ID` | Build timestamp |
| `JOB_NAME` | Name of the job |
| `WORKSPACE` | Workspace directory |
| `GIT_COMMIT` | Git commit hash |
| `GIT_BRANCH` | Git branch name |

### Build Trigger Cron Syntax

| Expression | Meaning |
|------------|---------|
| `H/15 * * * *` | Every 15 minutes |
| `H 2 * * *` | Daily at 2 AM |
| `H 2 * * 1-5` | Weekdays at 2 AM |
| `@hourly` | Every hour |
| `@daily` | Once a day |

---

**Previous:** [05-jenkins-credentials-secrets.md](05-jenkins-credentials-secrets.md) | **Next:** [07-jenkins-pipelines.md](07-jenkins-pipelines.md)
