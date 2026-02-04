# Jenkins Configuration

## Initial Admin Password

```bash
# Get initial password after installation
# Linux
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# Docker
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword

# Windows
type "C:\Program Files\Jenkins\secrets\initialAdminPassword"
```

## User Management

### Create Users

```
Manage Jenkins → Users → Create User

Fields:
• Username
• Password
• Confirm password
• Full name
• Email address
```

### User Permissions with Matrix Authorization

```
Manage Jenkins → Security → Authorization → Matrix-based security

┌─────────────────────────────────────────────────────────────────┐
│                    Permission Matrix                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   User/Group    Overall  Job     View    SCM    Credentials    │
│   ────────────────────────────────────────────────────────      │
│   admin         ✓ All    ✓ All   ✓ All   ✓ All  ✓ All         │
│   developer     Read     Build   Read    Read   View           │
│   viewer        Read     Read    Read    -      -              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Role-Based Access Control

```
1. Install "Role-based Authorization Strategy" plugin
2. Manage Jenkins → Security → Authorization → Role-Based Strategy
3. Manage Jenkins → Manage and Assign Roles

Global Roles:
• admin - All permissions
• developer - Build, read permissions
• viewer - Read only

Project Roles:
• frontend-dev - Access to frontend-* jobs
• backend-dev - Access to backend-* jobs
```

## Security Configuration

### Configure Global Security

```
Manage Jenkins → Security

┌─────────────────────────────────────────────────────────────────┐
│                    Security Settings                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Security Realm (Authentication)                               │
│   ├── Jenkins' own user database                                │
│   ├── LDAP                                                      │
│   ├── Active Directory                                          │
│   └── Unix user/group database                                  │
│                                                                  │
│   Authorization                                                 │
│   ├── Anyone can do anything (not recommended)                  │
│   ├── Logged-in users can do anything                          │
│   ├── Matrix-based security                                     │
│   ├── Project-based Matrix Authorization                       │
│   └── Role-Based Strategy                                      │
│                                                                  │
│   Other Settings                                                │
│   ├── Enable CSRF Protection ✓                                 │
│   ├── Enable Agent → Controller Access Control ✓               │
│   └── Disable CLI over Remoting ✓                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### LDAP Configuration

```
Security Realm: LDAP

Server: ldap://ldap.example.com
Root DN: dc=example,dc=com
User search base: ou=users
User search filter: uid={0}
Group search base: ou=groups
Manager DN: cn=admin,dc=example,dc=com
Manager Password: ****
```

## System Configuration

### Configure System Settings

```
Manage Jenkins → System

Important Settings:
• Jenkins URL: http://jenkins.example.com:8080/
• System Admin email: admin@example.com
• # of executors: 2 (on controller)
• Usage: Only build jobs with matching labels
• Quiet period: 5 seconds
• SCM checkout retry count: 3
```

### Environment Variables

```
Manage Jenkins → System → Global properties → Environment variables

Add:
• JAVA_HOME = /usr/lib/jvm/java-17-openjdk
• MAVEN_HOME = /opt/maven
• PATH+EXTRA = $MAVEN_HOME/bin
```

### JDK Configuration

```
Manage Jenkins → Tools → JDK installations

Add JDK:
• Name: JDK17
• JAVA_HOME: /usr/lib/jvm/java-17-openjdk-amd64

Or auto-install:
• Install automatically ✓
• Install from adoptium.net
• Version: jdk-17.0.8+7
```

### Maven Configuration

```
Manage Jenkins → Tools → Maven installations

Add Maven:
• Name: Maven3
• MAVEN_HOME: /opt/maven

Or auto-install:
• Install automatically ✓
• Install from Apache
• Version: 3.9.4
```

### Git Configuration

```
Manage Jenkins → Tools → Git installations

Add Git:
• Name: Default
• Path to Git executable: /usr/bin/git

Or auto-install:
• Install automatically ✓
```

## Plugin Management

### Install Plugins

```
Manage Jenkins → Plugins → Available plugins

Essential Plugins:
• Git
• Pipeline
• Blue Ocean
• Docker Pipeline
• Kubernetes
• Credentials Binding
• SSH Agent
• Role-based Authorization Strategy
• Matrix Authorization Strategy
• Job DSL
• Build Timeout
• Timestamper
• AnsiColor
• Slack Notification (optional)
```

### Install from CLI

```bash
# Using Jenkins CLI
java -jar jenkins-cli.jar -s http://localhost:8080/ install-plugin git pipeline-stage-view

# Restart after installation
java -jar jenkins-cli.jar -s http://localhost:8080/ safe-restart
```

### Plugin Management via Script

```groovy
// In Script Console (Manage Jenkins → Script Console)
import jenkins.model.*

def plugins = ["git", "workflow-aggregator", "docker-workflow"]
def instance = Jenkins.getInstance()
def pm = instance.getPluginManager()
def uc = instance.getUpdateCenter()

plugins.each { plugin ->
    if (!pm.getPlugin(plugin)) {
        def p = uc.getPlugin(plugin)
        if (p) {
            p.deploy()
            println "Installing: ${plugin}"
        }
    }
}
```

## Email Configuration

### SMTP Configuration

```
Manage Jenkins → System → E-mail Notification

SMTP server: smtp.gmail.com
Default user e-mail suffix: @example.com

Advanced:
• Use SMTP Authentication ✓
• User Name: jenkins@example.com
• Password: ****
• Use SSL ✓
• SMTP Port: 465
```

### Extended Email Publisher

```
Manage Jenkins → System → Extended E-mail Notification

SMTP server: smtp.gmail.com
Default Content Type: HTML
Default Recipients: team@example.com
Default Subject: $PROJECT_NAME - Build #$BUILD_NUMBER - $BUILD_STATUS
Default Content: ${JELLY_SCRIPT,template="html"}
```

## Node Configuration

### Configure Executors

```
Manage Jenkins → Nodes → Built-In Node → Configure

# of executors: 2
Labels: master controller
Usage: Only build jobs with label expressions matching this node
```

### Disable Builds on Controller

```
# Best practice: Don't run builds on controller
# of executors: 0
Usage: Only build jobs with label expressions matching this node

# All builds run on agents
```

## Jenkins URL and Proxy

### Configure Jenkins URL

```
Manage Jenkins → System → Jenkins Location

Jenkins URL: http://jenkins.example.com:8080/
System Admin e-mail address: admin@example.com
```

### Configure Proxy

```
Manage Jenkins → Plugins → Advanced settings

HTTP Proxy Configuration:
Server: proxy.example.com
Port: 8080
User name: proxyuser
Password: ****
No Proxy Host: localhost, 127.0.0.1, *.internal.com
```

## Scriptler and Groovy

### Script Console

```
Manage Jenkins → Script Console

// List all jobs
Jenkins.instance.getAllItems(Job.class).each {
    println it.fullName
}

// List installed plugins
Jenkins.instance.pluginManager.plugins.each {
    println "${it.getShortName()}: ${it.getVersion()}"
}

// Cancel all queued builds
Jenkins.instance.queue.clear()

// Get system info
println "Jenkins Version: ${Jenkins.instance.getVersion()}"
println "Java Version: ${System.getProperty('java.version')}"
```

## Configuration as Code (JCasC)

### jenkins.yaml

```yaml
# /var/lib/jenkins/jenkins.yaml
jenkins:
  systemMessage: "Jenkins configured via JCasC"
  numExecutors: 0
  mode: EXCLUSIVE

  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: "admin"
          password: "${JENKINS_ADMIN_PASSWORD}"

  authorizationStrategy:
    loggedInUsersCanDoAnything:
      allowAnonymousRead: false

  globalNodeProperties:
    - envVars:
        env:
          - key: "JAVA_HOME"
            value: "/usr/lib/jvm/java-17-openjdk"

tool:
  git:
    installations:
      - name: "Default"
        home: "/usr/bin/git"

  maven:
    installations:
      - name: "Maven3"
        properties:
          - installSource:
              installers:
                - maven:
                    id: "3.9.4"

unclassified:
  location:
    url: "http://jenkins.example.com:8080/"
    adminAddress: "admin@example.com"
```

### Enable JCasC

```bash
# Set environment variable
export CASC_JENKINS_CONFIG=/var/lib/jenkins/jenkins.yaml

# Or in /etc/default/jenkins
JAVA_ARGS="-Djenkins.install.runSetupWizard=false -Dcasc.jenkins.config=/var/lib/jenkins/jenkins.yaml"
```

## Quick Reference

### Important URLs

| URL | Description |
|-----|-------------|
| `/manage` | Management console |
| `/pluginManager` | Plugin management |
| `/credentials` | Credentials management |
| `/script` | Script console |
| `/systemInfo` | System information |
| `/log/all` | All logs |

### Common Configuration Paths

| Setting | Location |
|---------|----------|
| Users | Manage Jenkins → Users |
| Security | Manage Jenkins → Security |
| Plugins | Manage Jenkins → Plugins |
| System | Manage Jenkins → System |
| Tools | Manage Jenkins → Tools |
| Nodes | Manage Jenkins → Nodes |

### Default Credentials Location

| Platform | Path |
|----------|------|
| Linux | `/var/lib/jenkins/secrets/initialAdminPassword` |
| Docker | `/var/jenkins_home/secrets/initialAdminPassword` |
| Windows | `C:\Program Files\Jenkins\secrets\initialAdminPassword` |

---

**Previous:** [03-jenkins-installation.md](03-jenkins-installation.md) | **Next:** [05-jenkins-credentials-secrets.md](05-jenkins-credentials-secrets.md)
