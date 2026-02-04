# Jenkins Credentials and Secrets

## Credentials Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                  Jenkins Credentials System                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Credentials Store                                             │
│   ├── System (Jenkins Controller)                               │
│   │   └── Global credentials (unrestricted)                    │
│   │                                                              │
│   ├── Folder-level credentials                                  │
│   │   └── Available to jobs in folder                          │
│   │                                                              │
│   └── User credentials                                          │
│       └── Per-user credentials                                  │
│                                                                  │
│   Credential Types:                                             │
│   • Username with password                                      │
│   • SSH Username with private key                               │
│   • Secret text                                                 │
│   • Secret file                                                 │
│   • Certificate                                                 │
│   • Docker Host Certificate Authentication                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Credential Types

### Username with Password

```
Manage Jenkins → Credentials → System → Global credentials → Add Credentials

Kind: Username with password
Scope: Global
Username: deploy-user
Password: ****
ID: deploy-credentials
Description: Deployment credentials
```

### SSH Username with Private Key

```
Kind: SSH Username with private key
Scope: Global
ID: github-ssh-key
Description: GitHub SSH Key
Username: git
Private Key: Enter directly
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA...
-----END RSA PRIVATE KEY-----
Passphrase: (optional)
```

### Secret Text

```
Kind: Secret text
Scope: Global
Secret: my-super-secret-api-key
ID: api-key
Description: External API Key
```

### Secret File

```
Kind: Secret file
Scope: Global
File: Upload file (e.g., kubeconfig, certificate)
ID: kubeconfig
Description: Kubernetes config file
```

### Certificate

```
Kind: Certificate
Scope: Global
Certificate: Upload PKCS#12 file
Password: ****
ID: ssl-cert
Description: SSL Certificate
```

## Using Credentials in Freestyle Jobs

### Source Code Management

```
Source Code Management → Git
Repository URL: git@github.com:user/repo.git
Credentials: Select "github-ssh-key"
```

### Build Environment

```
Build Environment → Use secret text(s) or file(s)

Bindings:
├── Secret text
│   Variable: API_KEY
│   Credentials: api-key
│
├── Username and password (separated)
│   Username Variable: DEPLOY_USER
│   Password Variable: DEPLOY_PASS
│   Credentials: deploy-credentials
│
└── SSH User Private Key
    Key File Variable: SSH_KEY_FILE
    Credentials: github-ssh-key
```

## Using Credentials in Pipelines

### withCredentials Block

```groovy
pipeline {
    agent any
    stages {
        stage('Deploy') {
            steps {
                // Username and password
                withCredentials([usernamePassword(
                    credentialsId: 'deploy-credentials',
                    usernameVariable: 'USERNAME',
                    passwordVariable: 'PASSWORD'
                )]) {
                    sh '''
                        echo "Deploying as $USERNAME"
                        curl -u $USERNAME:$PASSWORD https://api.example.com/deploy
                    '''
                }
            }
        }
    }
}
```

### Secret Text

```groovy
pipeline {
    agent any
    stages {
        stage('API Call') {
            steps {
                withCredentials([string(
                    credentialsId: 'api-key',
                    variable: 'API_KEY'
                )]) {
                    sh '''
                        curl -H "Authorization: Bearer $API_KEY" \
                            https://api.example.com/data
                    '''
                }
            }
        }
    }
}
```

### SSH Key

```groovy
pipeline {
    agent any
    stages {
        stage('SSH Deploy') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'server-ssh-key',
                    keyFileVariable: 'SSH_KEY',
                    usernameVariable: 'SSH_USER'
                )]) {
                    sh '''
                        ssh -i $SSH_KEY $SSH_USER@server.example.com \
                            "cd /app && ./deploy.sh"
                    '''
                }
            }
        }
    }
}
```

### Secret File

```groovy
pipeline {
    agent any
    stages {
        stage('Kubernetes Deploy') {
            steps {
                withCredentials([file(
                    credentialsId: 'kubeconfig',
                    variable: 'KUBECONFIG'
                )]) {
                    sh '''
                        kubectl --kubeconfig=$KUBECONFIG apply -f deployment.yaml
                    '''
                }
            }
        }
    }
}
```

### Multiple Credentials

```groovy
pipeline {
    agent any
    stages {
        stage('Deploy') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-hub',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    ),
                    string(
                        credentialsId: 'slack-webhook',
                        variable: 'SLACK_WEBHOOK'
                    ),
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    sh '''
                        docker login -u $DOCKER_USER -p $DOCKER_PASS
                        docker push myapp:latest
                        kubectl --kubeconfig=$KUBECONFIG apply -f k8s/
                        curl -X POST $SLACK_WEBHOOK -d '{"text":"Deployed!"}'
                    '''
                }
            }
        }
    }
}
```

## Environment Directive

```groovy
pipeline {
    agent any
    environment {
        // Bind credentials to environment variables
        AWS_CREDENTIALS = credentials('aws-credentials')
        // Creates AWS_CREDENTIALS_USR and AWS_CREDENTIALS_PSW

        API_KEY = credentials('api-key')
        // For secret text, creates API_KEY directly
    }
    stages {
        stage('Deploy to AWS') {
            steps {
                sh '''
                    aws configure set aws_access_key_id $AWS_CREDENTIALS_USR
                    aws configure set aws_secret_access_key $AWS_CREDENTIALS_PSW
                    aws s3 sync ./dist s3://my-bucket/
                '''
            }
        }
    }
}
```

## Credential Scopes

| Scope | Description |
|-------|-------------|
| **Global** | Available everywhere |
| **System** | Only available to Jenkins system (agents, etc.) |
| **User** | Only for specific user |

```
┌─────────────────────────────────────────────────────────────────┐
│                   Credential Scopes                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Global (unrestricted)                                         │
│   └── Available to all jobs and pipelines                       │
│                                                                  │
│   System                                                        │
│   └── Jenkins internal use (connecting to agents)              │
│                                                                  │
│   Folder-level                                                  │
│   └── Available only within specific folder                    │
│       └── Useful for team/project isolation                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Credentials via CLI

### Create Credentials with jenkins-cli

```bash
# Create credentials XML
cat > cred.xml << 'EOF'
<com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl>
  <scope>GLOBAL</scope>
  <id>my-credentials</id>
  <description>My Credentials</description>
  <username>admin</username>
  <password>secret123</password>
</com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl>
EOF

# Create credential
java -jar jenkins-cli.jar -s http://localhost:8080/ \
    create-credentials-by-xml system::system::jenkins _ < cred.xml
```

### List Credentials

```bash
java -jar jenkins-cli.jar -s http://localhost:8080/ \
    list-credentials system::system::jenkins
```

## Credentials via Groovy Script

### Create Username/Password Credential

```groovy
import jenkins.model.*
import com.cloudbees.plugins.credentials.*
import com.cloudbees.plugins.credentials.common.*
import com.cloudbees.plugins.credentials.domains.*
import com.cloudbees.plugins.credentials.impl.*

def domain = Domain.global()
def store = Jenkins.instance.getExtensionList(
    'com.cloudbees.plugins.credentials.SystemCredentialsProvider'
)[0].getStore()

def credentials = new UsernamePasswordCredentialsImpl(
    CredentialsScope.GLOBAL,
    "my-cred-id",
    "Description",
    "username",
    "password"
)

store.addCredentials(domain, credentials)
```

### Create Secret Text Credential

```groovy
import org.jenkinsci.plugins.plaincredentials.impl.StringCredentialsImpl
import hudson.util.Secret

def credentials = new StringCredentialsImpl(
    CredentialsScope.GLOBAL,
    "api-key",
    "API Key",
    Secret.fromString("my-secret-value")
)

store.addCredentials(domain, credentials)
```

## Credentials Security

### Best Practices

```
┌─────────────────────────────────────────────────────────────────┐
│              Credentials Security Best Practices                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ✓ Use folder-level credentials for isolation                  │
│   ✓ Rotate credentials regularly                                │
│   ✓ Use meaningful IDs and descriptions                        │
│   ✓ Limit credential access with RBAC                          │
│   ✓ Audit credential usage                                      │
│   ✓ Never log credential values                                 │
│   ✓ Use credential binding, not inline secrets                  │
│                                                                  │
│   ✗ Don't store credentials in source code                      │
│   ✗ Don't echo credentials in build logs                        │
│   ✗ Don't use global scope unnecessarily                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Mask Passwords in Logs

```groovy
pipeline {
    agent any
    options {
        // Mask passwords in console output
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }
    stages {
        stage('Deploy') {
            steps {
                // Credentials are automatically masked
                withCredentials([usernamePassword(
                    credentialsId: 'deploy-creds',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    // $PASS will show as **** in logs
                    sh 'echo "Password is $PASS"'
                }
            }
        }
    }
}
```

## External Secret Management

### HashiCorp Vault Integration

```groovy
// Install HashiCorp Vault Plugin
pipeline {
    agent any
    stages {
        stage('Get Secrets') {
            steps {
                withVault(
                    configuration: [
                        vaultUrl: 'https://vault.example.com',
                        vaultCredentialId: 'vault-token'
                    ],
                    vaultSecrets: [
                        [
                            path: 'secret/myapp',
                            secretValues: [
                                [envVar: 'DB_PASSWORD', vaultKey: 'password']
                            ]
                        ]
                    ]
                ) {
                    sh 'echo "Using secret from Vault"'
                }
            }
        }
    }
}
```

## Quick Reference

### Credential Types

| Type | Use Case |
|------|----------|
| Username/Password | API auth, Git over HTTPS |
| SSH Key | Git over SSH, server access |
| Secret Text | API keys, tokens |
| Secret File | Kubeconfig, certificates |
| Certificate | SSL/TLS, code signing |

### Pipeline Bindings

| Type | Variables Created |
|------|-------------------|
| `usernamePassword` | `_USR`, `_PSW` |
| `string` | Direct variable |
| `file` | File path variable |
| `sshUserPrivateKey` | `keyFileVariable`, `usernameVariable` |

### Common Credential IDs

| ID Pattern | Example |
|------------|---------|
| `<provider>-<purpose>` | `github-ssh-key` |
| `<env>-<service>` | `prod-database-creds` |
| `<team>-<resource>` | `devops-aws-credentials` |

---

**Previous:** [04-jenkins-configuration.md](04-jenkins-configuration.md) | **Next:** [06-jenkins-freestyle-jobs.md](06-jenkins-freestyle-jobs.md)
