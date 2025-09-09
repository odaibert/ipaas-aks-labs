# Simplifying API Management on AKS with Azure API Management & API Center

This comprehensive guide will show you how to deploy a modern API management solution using Azure Kubernetes Service (AKS), Azure API Management (APIM), and Azure API Center. You'll learn to expose, secure, document, and govern APIs running on AKS without complex dependencies or external tools.

## Objectives

After completing this workshop, you will be able to:

• Deploy a sample API to an AKS cluster
• Set up Azure API Management to expose and secure your APIs
• Register APIs in Azure API Center for discovery and governance
• Configure authentication, rate limiting, and monitoring
• Troubleshoot common API management issues
• Clean up all resources when finished

## Prerequisites

Before you begin, you will need an [Azure subscription](https://azure.microsoft.com/) with Contributor permissions and basic familiarity with Azure Portal.

In addition, you will need the following tools installed on your local machine:

• [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli?view=azure-cli-latest)
• PowerShell (already available on Windows)

### Setup Azure CLI

Start by logging into Azure by running the following command and follow the prompts:

```powershell
az login --use-device-code
```

> **TIP**
> You can log into a different tenant by passing in the `--tenant` flag to specify your tenant domain or tenant ID.

Register required resource providers for this workshop:

```powershell
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.ApiManagement
az provider register --namespace Microsoft.ApiCenter
```

Check the status of the resource provider registration:

```powershell
az provider show --namespace Microsoft.ContainerService --query "registrationState"
az provider show --namespace Microsoft.ApiManagement --query "registrationState"
az provider show --namespace Microsoft.ApiCenter --query "registrationState"
```

> **WARNING**
> Make sure all providers show "Registered" status before proceeding.

### Setup Resource Group

In this workshop, we will set environment variables for the resource group name and location.

> **IMPORTANT**
> The following commands will set the environment variables for your current PowerShell session. If you close the terminal, you will need to set the variables again.

To keep resource names unique, we will use a random number as a suffix:

```powershell
$RAND = Get-Random -Maximum 9999
$env:RAND = $RAND
Write-Host "Random resource identifier will be: $RAND"
```

Set the location to a region of your choice. For example, `eastus` or `westeurope`:

```powershell
$env:LOCATION = "eastus"
```

Create a resource group name using the random number:

```powershell
$env:RG_NAME = "apim-aks-lab-$RAND"
```

> **TIP**
> You can list available regions with:
> ```powershell
> az account list-locations --query "[].{Region:name}" --output table
> ```

Run the following command to create a resource group:

```powershell
az group create --name $env:RG_NAME --location $env:LOCATION
```

## Deploy Infrastructure Components

### Create AKS Cluster

First, let's create an AKS cluster to host our sample API:

```powershell
az aks create `
  --resource-group $env:RG_NAME `
  --name "aks-cluster-$env:RAND" `
  --node-count 2 `
  --node-vm-size Standard_B2s `
  --enable-managed-identity `
  --generate-ssh-keys `
  --enable-addons monitoring
```

> **INFO**
> This will take 5-10 minutes to complete. The cluster will be created with monitoring enabled and managed identity for secure Azure service integration.

Get credentials for the AKS cluster:

```powershell
az aks get-credentials --resource-group $env:RG_NAME --name "aks-cluster-$env:RAND"
```

### Create Azure API Management Instance

Create an API Management instance to act as our API gateway:

```powershell
az apim create `
  --resource-group $env:RG_NAME `
  --name "apim-$env:RAND" `
  --publisher-email "admin@contoso.com" `
  --publisher-name "Contoso API Management" `
  --sku-name Developer
```

> **WARNING**
> This process can take 30-45 minutes to complete. Do not close the terminal while this is running.

### Create Azure API Center

Create an API Center for API discovery and governance:

```powershell
az apic create `
  --resource-group $env:RG_NAME `
  --name "apic-$env:RAND" `
  --location $env:LOCATION
```

## Deploy Sample API to AKS

### Create Sample API Application

We'll create a simple Node.js REST API with user management functionality. This API will demonstrate how to expose, secure, and manage APIs using Azure services.

#### Step 1: Create the Application Directory

First, let's create a dedicated folder for our sample API:

```powershell
# Create a new directory for our sample API
New-Item -ItemType Directory -Path "sample-api" -Force
Set-Location "sample-api"
Write-Host "✅ Created sample-api directory"
```

#### Step 2: Create the Package Configuration

Create a `package.json` file that defines our Node.js application dependencies:

```powershell
# Create package.json with application metadata and dependencies
@"
{
  "name": "sample-api",
  "version": "1.0.0",
  "description": "Sample API for AKS and APIM integration",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.0"
  }
}
"@ | Out-File -FilePath "package.json" -Encoding utf8
Write-Host "✅ Created package.json"
```

#### Step 3: Create the API Server Code

Create the main `server.js` file with our REST API endpoints:

```powershell
# Create the main server file with REST API endpoints
@"
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

// Enable JSON parsing for requests
app.use(express.json());

// Sample data - in production, this would be a database
let users = [
  { id: 1, name: 'John Doe', email: 'john@example.com' },
  { id: 2, name: 'Jane Smith', email: 'jane@example.com' }
];

// Health check endpoint - used for monitoring
app.get('/health', (req, res) => {
  res.status(200).json({ 
    status: 'healthy', 
    timestamp: new Date().toISOString(),
    version: '1.0.0'
  });
});

// REST API Endpoints for User Management

// GET /api/users - Get all users
app.get('/api/users', (req, res) => {
  console.log('GET /api/users - Fetching all users');
  res.json({
    users: users,
    count: users.length
  });
});

// GET /api/users/:id - Get specific user by ID
app.get('/api/users/:id', (req, res) => {
  const userId = parseInt(req.params.id);
  console.log(\`GET /api/users/\${userId} - Fetching user\`);
  
  const user = users.find(u => u.id === userId);
  if (!user) {
    return res.status(404).json({ 
      error: 'User not found',
      userId: userId 
    });
  }
  res.json(user);
});

// POST /api/users - Create a new user
app.post('/api/users', (req, res) => {
  console.log('POST /api/users - Creating new user');
  
  // Validate required fields
  if (!req.body.name || !req.body.email) {
    return res.status(400).json({
      error: 'Missing required fields: name and email'
    });
  }

  const newUser = {
    id: users.length + 1,
    name: req.body.name,
    email: req.body.email,
    created: new Date().toISOString()
  };
  
  users.push(newUser);
  res.status(201).json(newUser);
});

// Start the server
app.listen(port, () => {
  console.log(\`🚀 Sample API listening on port \${port}\`);
  console.log(\`📋 Available endpoints:\`);
  console.log(\`   GET  /health        - Health check\`);
  console.log(\`   GET  /api/users     - Get all users\`);
  console.log(\`   GET  /api/users/:id - Get user by ID\`);
  console.log(\`   POST /api/users     - Create new user\`);
});
"@ | Out-File -FilePath "server.js" -Encoding utf8
Write-Host "✅ Created server.js with REST API endpoints"
```

#### Step 4: Create the Dockerfile

Create a `Dockerfile` to containerize our application:

```powershell
# Create Dockerfile for containerizing the application
@"
# Use official Node.js runtime as base image
FROM node:18-alpine

# Set working directory inside container
WORKDIR /app

# Copy package files first (for better Docker layer caching)
COPY package*.json ./

# Install dependencies
RUN npm install --production

# Copy application code
COPY . .

# Expose port 3000
EXPOSE 3000

# Set non-root user for security
USER node

# Start the application
CMD ["npm", "start"]
"@ | Out-File -FilePath "Dockerfile" -Encoding utf8
Write-Host "✅ Created Dockerfile"
```

#### Step 5: Verify the Application Files

Let's confirm all files were created successfully:

```powershell
# List all files in the sample-api directory
Write-Host "📁 Files created in sample-api directory:"
Get-ChildItem -Name
Write-Host ""
Write-Host "✅ Sample API application is ready!"
Write-Host "📊 API Features:"
Write-Host "   - Health check endpoint for monitoring"
Write-Host "   - RESTful user management (GET, POST)"
Write-Host "   - JSON request/response handling"
Write-Host "   - Error handling and validation"
Write-Host "   - Containerized with Docker"
```

> **INFO**
> Our sample API includes:
> - **Health Check**: `/health` endpoint for monitoring and load balancer health checks
> - **User Management**: CRUD operations for managing users
> - **REST Standards**: Proper HTTP methods and status codes
> - **Error Handling**: Validation and meaningful error messages
> - **Logging**: Console output for debugging and monitoring

### Build and Push Container Image

Create Azure Container Registry and push our API image:

```powershell
# Create ACR
az acr create `
  --resource-group $env:RG_NAME `
  --name "acr$env:RAND" `
  --sku Basic `
  --admin-enabled true

# Get ACR login server
$ACR_LOGIN_SERVER = az acr show --name "acr$env:RAND" --resource-group $env:RG_NAME --query "loginServer" --output tsv

# Build and push image
az acr build --registry "acr$env:RAND" --image "sample-api:v1" .
```

### Deploy API to AKS

Create Kubernetes manifests for our API:

```powershell
# Create deployment manifest
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-api
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-api
  template:
    metadata:
      labels:
        app: sample-api
    spec:
      containers:
      - name: sample-api
        image: $ACR_LOGIN_SERVER/sample-api:v1
        ports:
        - containerPort: 3000
        env:
        - name: PORT
          value: "3000"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: sample-api-service
  namespace: default
spec:
  selector:
    app: sample-api
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
  type: LoadBalancer
"@ | Out-File -FilePath "k8s-deployment.yaml" -Encoding utf8

# Apply the deployment
kubectl apply -f k8s-deployment.yaml
```

Get the external IP of the service:

```powershell
# Wait for external IP (this may take a few minutes)
Write-Host "Waiting for external IP assignment..."
do {
    $EXTERNAL_IP = kubectl get service sample-api-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
    Start-Sleep -Seconds 10
    Write-Host "Checking for external IP..."
} while ([string]::IsNullOrEmpty($EXTERNAL_IP))

Write-Host "External IP assigned: $EXTERNAL_IP"
$env:API_EXTERNAL_IP = $EXTERNAL_IP
```

Test the API directly:

```powershell
# Test the API
Invoke-RestMethod -Uri "http://$env:API_EXTERNAL_IP/health" -Method GET
Invoke-RestMethod -Uri "http://$env:API_EXTERNAL_IP/api/users" -Method GET
```

## Configure Azure API Management

### Import API into APIM

Now let's expose our AKS API through Azure API Management:

```powershell
# Get APIM service URL
$APIM_URL = az apim show --name "apim-$env:RAND" --resource-group $env:RG_NAME --query "gatewayUrl" --output tsv

# Create OpenAPI specification for our API
@"
{
  "openapi": "3.0.0",
  "info": {
    "title": "Sample Users API",
    "version": "1.0.0",
    "description": "A sample API running on AKS"
  },
  "servers": [
    {
      "url": "http://$env:API_EXTERNAL_IP"
    }
  ],
  "paths": {
    "/health": {
      "get": {
        "summary": "Health check",
        "responses": {
          "200": {
            "description": "API is healthy"
          }
        }
      }
    },
    "/api/users": {
      "get": {
        "summary": "Get all users",
        "responses": {
          "200": {
            "description": "List of users"
          }
        }
      },
      "post": {
        "summary": "Create a new user",
        "requestBody": {
          "required": true,
          "content": {
            "application/json": {
              "schema": {
                "type": "object",
                "properties": {
                  "name": {
                    "type": "string"
                  },
                  "email": {
                    "type": "string"
                  }
                }
              }
            }
          }
        },
        "responses": {
          "201": {
            "description": "User created"
          }
        }
      }
    },
    "/api/users/{id}": {
      "get": {
        "summary": "Get user by ID",
        "parameters": [
          {
            "name": "id",
            "in": "path",
            "required": true,
            "schema": {
              "type": "integer"
            }
          }
        ],
        "responses": {
          "200": {
            "description": "User details"
          },
          "404": {
            "description": "User not found"
          }
        }
      }
    }
  }
}
"@ | Out-File -FilePath "api-spec.json" -Encoding utf8

# Import API into APIM
az apim api import `
  --resource-group $env:RG_NAME `
  --service-name "apim-$env:RAND" `
  --api-id "sample-users-api" `
  --path "users" `
  --specification-format OpenApi `
  --specification-path "api-spec.json" `
  --display-name "Sample Users API"
```

### Configure API Policies

Add rate limiting and other policies to secure the API:

```powershell
# Create policy XML for rate limiting and CORS
@"
<policies>
    <inbound>
        <base />
        <cors allow-credentials="false">
            <allowed-origins>
                <origin>*</origin>
            </allowed-origins>
            <allowed-methods>
                <method>GET</method>
                <method>POST</method>
            </allowed-methods>
            <allowed-headers>
                <header>*</header>
            </allowed-headers>
        </cors>
        <rate-limit-by-key calls="100" renewal-period="60" counter-key="@(context.Request.IpAddress)" />
        <set-header name="X-Powered-By" exists-action="override">
            <value>Azure API Management</value>
        </set-header>
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
        <set-header name="X-Response-Time" exists-action="override">
            <value>@(context.Response.Headers.GetValueOrDefault("Date",""))</value>
        </set-header>
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
"@ | Out-File -FilePath "api-policy.xml" -Encoding utf8

# Apply the policy
az apim api policy create `
  --resource-group $env:RG_NAME `
  --service-name "apim-$env:RAND" `
  --api-id "sample-users-api" `
  --policy-content (Get-Content -Path "api-policy.xml" -Raw)
```

### Test API through APIM

Test the API through Azure API Management:

```powershell
# Get APIM gateway URL
$APIM_GATEWAY = az apim show --name "apim-$env:RAND" --resource-group $env:RG_NAME --query "gatewayUrl" --output tsv

Write-Host "Testing API through APIM Gateway: $APIM_GATEWAY"

# Test endpoints through APIM
Invoke-RestMethod -Uri "$APIM_GATEWAY/users/health" -Method GET
Invoke-RestMethod -Uri "$APIM_GATEWAY/users/api/users" -Method GET

Write-Host "API is successfully exposed through Azure API Management!"
```

## Register API in Azure API Center

### Create API in API Center

Register the API in Azure API Center for governance and discovery:

```powershell
# Create workspace in API Center
az apic workspace create `
  --resource-group $env:RG_NAME `
  --service-name "apic-$env:RAND" `
  --workspace-name "main" `
  --title "Main API Workspace"

# Register the API
az apic api create `
  --resource-group $env:RG_NAME `
  --service-name "apic-$env:RAND" `
  --workspace-name "main" `
  --api-name "sample-users-api" `
  --title "Sample Users API" `
  --kind "rest" `
  --description "A sample REST API for managing users, deployed on AKS and exposed through APIM"

# Create API version
az apic api version create `
  --resource-group $env:RG_NAME `
  --service-name "apic-$env:RAND" `
  --workspace-name "main" `
  --api-name "sample-users-api" `
  --version-name "v1" `
  --title "Version 1.0" `
  --lifecycle-stage "production"

# Create API definition (OpenAPI spec)
az apic api definition create `
  --resource-group $env:RG_NAME `
  --service-name "apic-$env:RAND" `
  --workspace-name "main" `
  --api-name "sample-users-api" `
  --version-name "v1" `
  --definition-name "openapi" `
  --title "OpenAPI Definition" `
  --description "OpenAPI 3.0 specification"

# Import the OpenAPI specification
az apic api definition import-specification `
  --resource-group $env:RG_NAME `
  --service-name "apic-$env:RAND" `
  --workspace-name "main" `
  --api-name "sample-users-api" `
  --version-name "v1" `
  --definition-name "openapi" `
  --format "link" `
  --value "$APIM_GATEWAY/users?format=openapi-link&api-version=2023-05-01-preview" `
  --specification '{"name":"openapi","version":"3.0.0"}'
```

### Add API Documentation

Create comprehensive documentation for the API:

```powershell
# Add environment information
az apic environment create `
  --resource-group $env:RG_NAME `
  --service-name "apic-$env:RAND" `
  --workspace-name "main" `
  --environment-name "production" `
  --title "Production Environment" `
  --description "Production environment hosted on AKS" `
  --kind "production"

# Link API deployment to environment
az apic api deployment create `
  --resource-group $env:RG_NAME `
  --service-name "apic-$env:RAND" `
  --workspace-name "main" `
  --api-name "sample-users-api" `
  --deployment-name "production-deployment" `
  --title "Production Deployment" `
  --description "API deployed on AKS cluster exposed through APIM" `
  --environment-name "production" `
  --definition-name "openapi" `
  --server '{"runtime-uri":"'$APIM_GATEWAY'/users"}'
```

## Monitoring and Observability

### Enable APIM Analytics

Configure monitoring and analytics for your APIs:

```powershell
# Enable Application Insights for APIM (optional)
$APP_INSIGHTS_NAME = "apim-insights-$env:RAND"

# Create Application Insights
az monitor app-insights component create `
  --app $APP_INSIGHTS_NAME `
  --location $env:LOCATION `
  --resource-group $env:RG_NAME `
  --kind web

# Get instrumentation key
$INSTRUMENTATION_KEY = az monitor app-insights component show `
  --app $APP_INSIGHTS_NAME `
  --resource-group $env:RG_NAME `
  --query "instrumentationKey" `
  --output tsv

Write-Host "Application Insights configured with key: $INSTRUMENTATION_KEY"
```

### View API Analytics

Check API usage and performance:

```powershell
# Generate some test traffic
Write-Host "Generating test traffic..."
for ($i = 1; $i -le 10; $i++) {
    try {
        Invoke-RestMethod -Uri "$APIM_GATEWAY/users/api/users" -Method GET | Out-Null
        Invoke-RestMethod -Uri "$APIM_GATEWAY/users/health" -Method GET | Out-Null
        Start-Sleep -Seconds 2
    } catch {
        Write-Host "Request $i failed: $($_.Exception.Message)"
    }
}

Write-Host "Test traffic generation complete. Check Azure Portal for analytics."
```

## Troubleshooting Common Issues

### Check API Health

Verify that all components are working correctly:

```powershell
Write-Host "=== System Health Check ==="

# Check AKS cluster status
Write-Host "1. Checking AKS cluster..."
$AKS_STATUS = az aks show --name "aks-cluster-$env:RAND" --resource-group $env:RG_NAME --query "provisioningState" --output tsv
Write-Host "   AKS Cluster Status: $AKS_STATUS"

# Check pods status
Write-Host "2. Checking pod status..."
kubectl get pods -o wide

# Check services
Write-Host "3. Checking services..."
kubectl get services

# Check APIM status
Write-Host "4. Checking APIM status..."
$APIM_STATUS = az apim show --name "apim-$env:RAND" --resource-group $env:RG_NAME --query "provisioningState" --output tsv
Write-Host "   APIM Status: $APIM_STATUS"

# Test direct API access
Write-Host "5. Testing direct API access..."
try {
    $DIRECT_RESPONSE = Invoke-RestMethod -Uri "http://$env:API_EXTERNAL_IP/health" -Method GET -TimeoutSec 10
    Write-Host "   Direct API Access: OK"
} catch {
    Write-Host "   Direct API Access: FAILED - $($_.Exception.Message)"
}

# Test APIM access
Write-Host "6. Testing APIM access..."
try {
    $APIM_RESPONSE = Invoke-RestMethod -Uri "$APIM_GATEWAY/users/health" -Method GET -TimeoutSec 10
    Write-Host "   APIM Access: OK"
} catch {
    Write-Host "   APIM Access: FAILED - $($_.Exception.Message)"
}

Write-Host "=== Health Check Complete ==="
```

### Common Issues and Solutions

```powershell
Write-Host "=== Troubleshooting Guide ==="
Write-Host ""
Write-Host "Common issues and solutions:"
Write-Host ""
Write-Host "1. External IP not assigned:"
Write-Host "   - Wait 5-10 minutes for Azure Load Balancer provisioning"
Write-Host "   - Check: kubectl get service sample-api-service -w"
Write-Host ""
Write-Host "2. APIM not responding:"
Write-Host "   - APIM deployment takes 30-45 minutes"
Write-Host "   - Check: az apim show --name apim-$env:RAND --resource-group $env:RG_NAME"
Write-Host ""
Write-Host "3. API not accessible through APIM:"
Write-Host "   - Verify backend service URL is correct"
Write-Host "   - Check APIM API policy configuration"
Write-Host "   - Verify AKS service external IP is accessible"
Write-Host ""
Write-Host "4. Rate limiting errors:"
Write-Host "   - Current limit: 100 calls per minute per IP"
Write-Host "   - Wait 1 minute or modify policy in Azure Portal"
```

## View Resources in Azure Portal

Access your deployed resources through the Azure Portal:

```powershell
Write-Host "=== Azure Portal Access ==="
Write-Host ""
Write-Host "Resource Group: $env:RG_NAME"
Write-Host "APIM Gateway URL: $APIM_GATEWAY"
Write-Host "Direct API URL: http://$env:API_EXTERNAL_IP"
Write-Host ""
Write-Host "Azure Portal URLs:"
Write-Host "- Resource Group: https://portal.azure.com/#@/resource/subscriptions/$(az account show --query 'id' -o tsv)/resourceGroups/$env:RG_NAME"
Write-Host "- API Management: https://portal.azure.com/#@/resource/subscriptions/$(az account show --query 'id' -o tsv)/resourceGroups/$env:RG_NAME/providers/Microsoft.ApiManagement/service/apim-$env:RAND"
Write-Host "- API Center: https://portal.azure.com/#@/resource/subscriptions/$(az account show --query 'id' -o tsv)/resourceGroups/$env:RG_NAME/providers/Microsoft.ApiCenter/services/apic-$env:RAND"
Write-Host "- AKS Cluster: https://portal.azure.com/#@/resource/subscriptions/$(az account show --query 'id' -o tsv)/resourceGroups/$env:RG_NAME/providers/Microsoft.ContainerService/managedClusters/aks-cluster-$env:RAND"
```

## Summary

In this comprehensive workshop, you successfully:

✅ **Deployed Infrastructure**: Created AKS cluster, Azure API Management, and Azure API Center
✅ **Built Sample API**: Developed a Node.js REST API and containerized it
✅ **Deployed to AKS**: Used Kubernetes to deploy and expose the API
✅ **Configured APIM**: Set up API gateway with policies for security and rate limiting
✅ **Registered in API Center**: Added API to centralized catalog for governance
✅ **Enabled Monitoring**: Configured analytics and health checks
✅ **Troubleshooting**: Learned to diagnose and resolve common issues

### Key Benefits Achieved

- **Centralized API Management**: All APIs accessible through a single gateway
- **Enhanced Security**: Rate limiting, CORS, and authentication policies applied
- **API Governance**: Centralized catalog with documentation and lifecycle management
- **Scalability**: Kubernetes-based deployment with Azure Load Balancer
- **Monitoring**: Built-in analytics and health monitoring
- **No Code Changes**: APIs integrated without modifying application code

### Architecture Overview

```
[Internet] → [Azure API Management] → [Azure Load Balancer] → [AKS Cluster]
                    ↓
            [Azure API Center]
                    ↓
          [Documentation & Governance]
```

### Next Steps

To extend this solution, consider:

• **Authentication**: Add Azure AD or OAuth 2.0 authentication
• **Advanced Policies**: Implement request/response transformation
• **Multiple Environments**: Set up staging and development environments
• **CI/CD Pipeline**: Automate deployment with Azure DevOps or GitHub Actions
• **Advanced Monitoring**: Add custom metrics and alerting
• **API Versioning**: Implement versioning strategies for your APIs

## Cleanup

> **WARNING**
> This will permanently delete all resources created in this workshop. Make sure you no longer need these resources.

To clean up all resources created in this workshop:

```powershell
Write-Host "Starting cleanup process..."

# Delete the resource group and all its contents
az group delete `
  --name $env:RG_NAME `
  --yes `
  --no-wait

Write-Host "Cleanup initiated. Resources are being deleted in the background."
Write-Host "This process may take 10-15 minutes to complete."
Write-Host ""
Write-Host "You can monitor the deletion progress in the Azure Portal:"
Write-Host "https://portal.azure.com/#blade/HubsExtension/BrowseResourceGroups"
```

### Cleanup Verification

To verify cleanup completion:

```powershell
# Check if resource group still exists
$RG_EXISTS = az group exists --name $env:RG_NAME
if ($RG_EXISTS -eq "true") {
    Write-Host "Resource group still exists. Cleanup in progress..."
} else {
    Write-Host "✅ Cleanup complete! All resources have been deleted."
}

# Clear environment variables
Remove-Variable -Name "RG_NAME" -ErrorAction SilentlyContinue
Remove-Variable -Name "LOCATION" -ErrorAction SilentlyContinue
Remove-Variable -Name "RAND" -ErrorAction SilentlyContinue
Remove-Variable -Name "API_EXTERNAL_IP" -ErrorAction SilentlyContinue

Write-Host "Environment variables cleared."
```

---

## Additional Resources

To learn more about the technologies used in this workshop:

• [Azure Kubernetes Service (AKS) documentation](https://learn.microsoft.com/azure/aks)
• [Azure API Management documentation](https://learn.microsoft.com/azure/api-management)
• [Azure API Center documentation](https://learn.microsoft.com/azure/api-center)
• [Kubernetes documentation](https://kubernetes.io/docs)
• [Docker documentation](https://docs.docker.com)

If you have feedback or suggestions for this workshop, please feel free to open an issue or pull request in the repository.

---

**Happy API Management!** 🚀
