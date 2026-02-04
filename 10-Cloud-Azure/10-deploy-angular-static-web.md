# Deploy Angular to Azure Static Web Apps

## What is Azure Static Web Apps?

Azure Static Web Apps is a service that automatically builds and deploys full-stack web apps from a GitHub or Azure DevOps repository.

```
┌─────────────────────────────────────────────────────────────┐
│              AZURE STATIC WEB APPS                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────┐ │
│  │  GitHub  │───▶│ GitHub       │───▶│ Azure Static     │ │
│  │  Push    │    │ Actions      │    │ Web Apps         │ │
│  └──────────┘    │ (Auto-built) │    │                  │ │
│                  └──────────────┘    │ ┌──────────────┐ │ │
│                                      │ │   Angular    │ │ │
│                                      │ │   (Static)   │ │ │
│                                      │ └──────────────┘ │ │
│                                      │        │        │ │
│                                      │        ▼        │ │
│                                      │ ┌──────────────┐ │ │
│                                      │ │  API (.NET)  │ │ │
│                                      │ │  (Optional)  │ │ │
│                                      │ └──────────────┘ │ │
│                                      └──────────────────┘ │
│                                                             │
│  Features:                                                  │
│  • Free SSL certificates         • Global CDN              │
│  • Staging environments          • Custom domains          │
│  • Integrated API (Functions)    • Authentication          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Pricing Tiers

| Feature | Free | Standard |
|---------|------|----------|
| **Bandwidth** | 100 GB/month | 100 GB included |
| **Custom domains** | 2 | 5 |
| **Staging environments** | 3 | 10 |
| **API (Functions)** | Managed | Bring your own |
| **SLA** | None | 99.95% |
| **Private endpoints** | No | Yes |

---

## Angular Project Setup

### Create Angular Project

```bash
# Create new Angular project
ng new my-angular-app --routing --style=scss
cd my-angular-app

# Generate components
ng generate component components/home
ng generate component components/about
ng generate service services/api
```

### Configure for Production

```typescript
// src/environments/environment.prod.ts
export const environment = {
  production: true,
  apiUrl: '/api'  // Static Web Apps proxies /api to backend
};
```

### API Service

```typescript
// src/app/services/api.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { environment } from '../../environments/environment';

@Injectable({
  providedIn: 'root'
})
export class ApiService {
  private apiUrl = environment.apiUrl;

  constructor(private http: HttpClient) {}

  getData() {
    return this.http.get(`${this.apiUrl}/data`);
  }
}
```

### Configure Angular for SPA Routing

```json
// staticwebapp.config.json (in project root)
{
  "navigationFallback": {
    "rewrite": "/index.html",
    "exclude": ["/assets/*", "/*.{css,js,png,jpg,svg,ico}"]
  },
  "routes": [
    {
      "route": "/api/*",
      "allowedRoles": ["authenticated"]
    }
  ],
  "responseOverrides": {
    "401": {
      "statusCode": 302,
      "redirect": "/.auth/login/aad"
    }
  },
  "globalHeaders": {
    "X-Content-Type-Options": "nosniff",
    "X-Frame-Options": "DENY",
    "Content-Security-Policy": "default-src 'self'"
  }
}
```

---

## Create Static Web App

### Using Azure Portal

1. Search "Static Web Apps"
2. Click "Create"
3. Select subscription and resource group
4. Enter name: `swa-myapp-prod`
5. Select region
6. Connect to GitHub repository
7. Configure build settings:
   - App location: `/`
   - API location: `api` (optional)
   - Output location: `dist/my-angular-app`

### Using Azure CLI

```bash
# Create Static Web App
az staticwebapp create \
  --name swa-myapp-prod \
  --resource-group rg-myapp-prod \
  --source https://github.com/myorg/my-angular-app \
  --location eastus2 \
  --branch main \
  --app-location "/" \
  --output-location "dist/my-angular-app" \
  --login-with-github

# List static web apps
az staticwebapp list -o table

# Get deployment token (for CI/CD)
az staticwebapp secrets list \
  --name swa-myapp-prod \
  --resource-group rg-myapp-prod \
  --query "properties.apiKey" -o tsv
```

---

## GitHub Actions Deployment

When you connect GitHub, a workflow is automatically created:

### Auto-generated Workflow

```yaml
# .github/workflows/azure-static-web-apps-xxx.yml
name: Azure Static Web Apps CI/CD

on:
  push:
    branches:
      - main
  pull_request:
    types: [opened, synchronize, reopened, closed]
    branches:
      - main

jobs:
  build_and_deploy_job:
    if: github.event_name == 'push' || (github.event_name == 'pull_request' && github.event.action != 'closed')
    runs-on: ubuntu-latest
    name: Build and Deploy Job
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true

      - name: Build And Deploy
        id: builddeploy
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN }}
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          action: "upload"
          app_location: "/"
          api_location: "api"
          output_location: "dist/my-angular-app"

  close_pull_request_job:
    if: github.event_name == 'pull_request' && github.event.action == 'closed'
    runs-on: ubuntu-latest
    name: Close Pull Request Job
    steps:
      - name: Close Pull Request
        id: closepullrequest
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN }}
          action: "close"
```

### Custom Workflow with Tests

```yaml
# .github/workflows/deploy-angular.yml
name: Deploy Angular to Static Web Apps

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Test
        run: npm run test -- --watch=false --browsers=ChromeHeadless

      - name: Build
        run: npm run build -- --configuration=production

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: angular-build
          path: dist/my-angular-app

  deploy:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    steps:
      - uses: actions/checkout@v4

      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: angular-build
          path: dist/my-angular-app

      - name: Deploy to Static Web Apps
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN }}
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          action: "upload"
          skip_app_build: true
          app_location: "dist/my-angular-app"
```

---

## Staging Environments

Static Web Apps automatically creates preview environments for PRs.

```
┌─────────────────────────────────────────────────────────────┐
│              STAGING ENVIRONMENTS                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Production:  https://swa-myapp-prod.azurestaticapps.net   │
│       ▲                                                     │
│       │ Merge PR                                            │
│       │                                                     │
│  PR #42:  https://swa-myapp-prod-42.azurestaticapps.net    │
│       ▲                                                     │
│       │ Open PR                                             │
│       │                                                     │
│  Feature Branch (local)                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Manage Environments

```bash
# List environments
az staticwebapp environment list \
  --name swa-myapp-prod \
  --resource-group rg-myapp-prod \
  -o table

# Delete staging environment
az staticwebapp environment delete \
  --name swa-myapp-prod \
  --resource-group rg-myapp-prod \
  --environment-name 42
```

---

## Custom Domain

### Add Custom Domain

```bash
# Add custom domain
az staticwebapp hostname set \
  --name swa-myapp-prod \
  --resource-group rg-myapp-prod \
  --hostname app.mycompany.com

# Validate domain
az staticwebapp hostname show \
  --name swa-myapp-prod \
  --resource-group rg-myapp-prod \
  --hostname app.mycompany.com
```

### DNS Configuration

```
# Add CNAME record to your DNS:
app.mycompany.com  CNAME  swa-myapp-prod.azurestaticapps.net

# For apex domain (mycompany.com), use ALIAS or ANAME record
# Or add TXT validation record first
```

---

## Connect to .NET API Backend

### Option 1: External API

```typescript
// environment.prod.ts
export const environment = {
  production: true,
  apiUrl: 'https://app-myapi-prod.azurewebsites.net/api'
};
```

### Option 2: Linked Backend (Bring Your Own Backend)

```bash
# Link App Service backend
az staticwebapp backends link \
  --name swa-myapp-prod \
  --resource-group rg-myapp-prod \
  --backend-resource-id /subscriptions/<sub-id>/resourceGroups/rg-myapp-prod/providers/Microsoft.Web/sites/app-myapi-prod \
  --backend-region eastus

# Requests to /api/* will be proxied to the App Service
```

### CORS Configuration (External API)

```bash
# Configure CORS on App Service
az webapp cors add \
  --name app-myapi-prod \
  --resource-group rg-myapp-prod \
  --allowed-origins "https://swa-myapp-prod.azurestaticapps.net" "https://app.mycompany.com"
```

---

## Authentication

### Built-in Auth Providers

```json
// staticwebapp.config.json
{
  "routes": [
    {
      "route": "/admin/*",
      "allowedRoles": ["admin"]
    },
    {
      "route": "/api/*",
      "allowedRoles": ["authenticated"]
    }
  ],
  "responseOverrides": {
    "401": {
      "statusCode": 302,
      "redirect": "/.auth/login/aad"
    }
  }
}
```

### Get User Info in Angular

```typescript
// src/app/services/auth.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';

interface UserInfo {
  identityProvider: string;
  userId: string;
  userDetails: string;
  userRoles: string[];
}

@Injectable({
  providedIn: 'root'
})
export class AuthService {
  constructor(private http: HttpClient) {}

  getUserInfo() {
    return this.http.get<{ clientPrincipal: UserInfo }>('/.auth/me');
  }

  login(provider: 'aad' | 'github' | 'google' = 'aad') {
    window.location.href = `/.auth/login/${provider}`;
  }

  logout() {
    window.location.href = '/.auth/logout';
  }
}
```

---

## Environment Variables

### Configure in Portal or CLI

```bash
# Set environment variable
az staticwebapp appsettings set \
  --name swa-myapp-prod \
  --resource-group rg-myapp-prod \
  --setting-names "API_URL=https://app-myapi-prod.azurewebsites.net"

# List settings
az staticwebapp appsettings list \
  --name swa-myapp-prod \
  --resource-group rg-myapp-prod
```

### Use in Angular (Build-time)

```typescript
// For build-time variables, use environment files
// environment.prod.ts is replaced during ng build --configuration=production
```

---

## Summary

| Feature | How To |
|---------|--------|
| Create SWA | `az staticwebapp create` |
| Deploy | Push to GitHub (auto-deploys) |
| Custom domain | `az staticwebapp hostname set` |
| Staging | Automatic PR preview environments |
| Connect API | Link backend or configure CORS |
| Auth | Built-in providers (AAD, GitHub) |

### Quick Commands

```bash
# Create
az staticwebapp create -n swa-name -g rg --source https://github.com/org/repo --branch main

# Get deployment token
az staticwebapp secrets list -n swa-name -g rg

# Add custom domain
az staticwebapp hostname set -n swa-name -g rg --hostname app.example.com

# Link backend API
az staticwebapp backends link -n swa-name -g rg --backend-resource-id <app-service-id>
```
