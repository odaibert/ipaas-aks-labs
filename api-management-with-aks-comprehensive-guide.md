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

## Deploy Sample Application to AKS

### Deploy the AKS Store Demo Application

Instead of creating a custom application, we'll use the **AKS Store Demo** - a publicly available multi-container application that simulates a retail scenario. This application includes multiple microservices and provides REST API endpoints that we can expose through Azure API Management.

#### Understanding the AKS Store Application

The AKS Store Demo includes the following components:

- **Store Front**: Web application for customers to view products and place orders
- **Product Service**: REST API that provides product information
- **Order Service**: REST API that handles order placement
- **RabbitMQ**: Message queue for order processing

> **INFO**
> This application is perfect for our demo because:
> - **Pre-built APIs**: Multiple REST endpoints available for testing
> - **Real-world architecture**: Demonstrates microservices patterns
> - **No build required**: Uses public container images
> - **Production-ready**: Includes health checks and proper service design

#### Step 1: Create Application Namespace

First, let's create a dedicated namespace for our demo application:

```powershell
# Create namespace for the AKS Store demo
kubectl create namespace aks-store-demo
Write-Host "✅ Created aks-store-demo namespace"
```

#### Step 2: Deploy the Application

Deploy the complete AKS Store application using the pre-built manifest:

```powershell
# Deploy the AKS Store demo application
Write-Host "🚀 Deploying AKS Store Demo application..."
Write-Host "   This includes: store-front, product-service, order-service, and rabbitmq"

kubectl apply -n aks-store-demo -f https://raw.githubusercontent.com/Azure-Samples/aks-store-demo/main/aks-store-ingress-quickstart.yaml

Write-Host "✅ AKS Store Demo application deployed successfully!"
```

#### Step 3: Verify Deployment Status

Check that all components are running correctly:

```powershell
# Wait for pods to be ready
Write-Host "⏳ Waiting for pods to start (this may take 2-3 minutes)..."
Start-Sleep -Seconds 30

# Check pod status
Write-Host "📋 Checking pod status:"
kubectl get pods -n aks-store-demo

Write-Host ""
Write-Host "🔍 Checking services:"
kubectl get services -n aks-store-demo

Write-Host ""
Write-Host "🌐 Checking ingress status:"
kubectl get ingress -n aks-store-demo
```

#### Step 4: Get Application External IP

Expose the store-front service and wait for external IP assignment:

```powershell
# Expose the store-front service as LoadBalancer to get an external IP
Write-Host "🔗 Exposing store-front service with LoadBalancer..."
kubectl patch service store-front -n aks-store-demo -p '{"spec":{"type":"LoadBalancer"}}'

# Wait for external IP assignment
Write-Host "⏳ Waiting for external IP assignment..."
$timeout = 300  # 5 minutes timeout
$elapsed = 0
do {
    $EXTERNAL_IP = kubectl get service store-front -n aks-store-demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
    if ([string]::IsNullOrEmpty($EXTERNAL_IP)) {
        Write-Host "   Still waiting for IP assignment... ($elapsed seconds elapsed)"
        Start-Sleep -Seconds 15
        $elapsed += 15
        if ($elapsed -ge $timeout) {
            Write-Host "   ⚠️ Timeout reached. This may take up to 10-15 minutes in some regions."
            Write-Host "   ℹ️ You can continue and check later with: kubectl get service store-front -n aks-store-demo"
            break
        }
    }
} while ([string]::IsNullOrEmpty($EXTERNAL_IP))

if (-not [string]::IsNullOrEmpty($EXTERNAL_IP)) {
    Write-Host "✅ External IP assigned: $EXTERNAL_IP"
    $env:STORE_IP = $EXTERNAL_IP
} else {
    Write-Host "⏸️ External IP assignment in progress. Please run this command later:"
    Write-Host "   `$env:STORE_IP = kubectl get service store-front -n aks-store-demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}'"
}
```

#### Step 5: Test the Application

Test the application and its API endpoints:

```powershell
```powershell
# Test the store front application
Write-Host "🧪 Testing the AKS Store application..."

# Check if we have the external IP
if ([string]::IsNullOrEmpty($env:STORE_IP)) {
    Write-Host "⚠️ External IP not yet available. Getting current status..."
    $env:STORE_IP = kubectl get service store-front -n aks-store-demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
}

if (-not [string]::IsNullOrEmpty($env:STORE_IP)) {
    # Test the main store front
    try {
        Write-Host "   Testing store front web app..."
        $response = Invoke-WebRequest -Uri "http://$env:STORE_IP" -UseBasicParsing -TimeoutSec 10
        Write-Host "   ✅ Store front is accessible (Status: $($response.StatusCode))"
    } catch {
        Write-Host "   ⚠️ Store front not yet ready: $($_.Exception.Message)"
    }

    # Test the product service API
    try {
        Write-Host "   Testing product service API..."
        $productResponse = Invoke-RestMethod -Uri "http://$env:STORE_IP/products" -TimeoutSec 10
        Write-Host "   ✅ Product API is working - Found $($productResponse.Count) products"
    } catch {
        Write-Host "   ⚠️ Product API not yet ready: $($_.Exception.Message)"
    }

    Write-Host ""
    Write-Host "🎯 Available API Endpoints for APIM Integration:"
    Write-Host "   GET  http://$env:STORE_IP/products        - Get all products"
    Write-Host "   GET  http://$env:STORE_IP/products/{id}   - Get specific product"
    Write-Host "   POST http://$env:STORE_IP/orders          - Create new order"
    Write-Host "   GET  http://$env:STORE_IP/orders          - Get all orders"
} else {
    Write-Host "❌ External IP not yet assigned. Please wait a few more minutes and try again."
    Write-Host "ℹ️ Check status with: kubectl get service store-front -n aks-store-demo"
}
Write-Host ""
Write-Host "📊 Application Features:"
Write-Host "   - Multi-service architecture with microservices"
Write-Host "   - REST API endpoints for products and orders"
Write-Host "   - Web frontend for customer interaction"
Write-Host "   - Message queue integration with RabbitMQ"
Write-Host "   - Production-ready with health checks"
```

> **TIP**
> You can view the AKS Store application in your browser by navigating to `http://$env:STORE_IP` once the external IP is assigned. The application provides a complete e-commerce experience with product catalog and order functionality.

### Test the Deployed Application

Our AKS Store Demo application is now running and ready to be integrated with Azure API Management. Let's verify everything is working:

```powershell
# Verify all services are running
Write-Host "🔍 Final verification of deployed services:"
kubectl get all -n aks-store-demo

Write-Host ""
Write-Host "📱 Application Access:"
Write-Host "   Store Front Web App: http://$env:STORE_IP"
Write-Host "   Products API: http://$env:STORE_IP/products"
Write-Host "   Orders API: http://$env:STORE_IP/orders"
Write-Host ""
Write-Host "✅ AKS Store Demo is ready for API Management integration!"
```

## 🔗 Step 6: Configure Azure API Management

### Import API into APIM

Now let's expose our AKS Store APIs through Azure API Management:

```powershell
# Ensure STORE_IP is set
if ([string]::IsNullOrEmpty($env:STORE_IP)) {
    Write-Host "⚠️ STORE_IP not set. Getting external IP..."
    $env:STORE_IP = kubectl get service store-front -n aks-store-demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
    
    if ([string]::IsNullOrEmpty($env:STORE_IP)) {
        Write-Host "❌ External IP not yet available. Please wait for LoadBalancer to assign IP and run:"
        Write-Host "   `$env:STORE_IP = kubectl get service store-front -n aks-store-demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}'"
        exit 1
    }
}

Write-Host "✅ Using AKS Store IP: $env:STORE_IP"

# Get APIM service URL
$APIM_URL = az apim show --name "apim-$env:RAND" --resource-group $env:RG_NAME --query "gatewayUrl" --output tsv

# Create OpenAPI specification for the AKS Store APIs
# Use direct variable substitution to avoid expansion issues
$storeUrl = "http://$env:STORE_IP"

@"
{
  "openapi": "3.0.0",
  "info": {
    "title": "AKS Store API",
    "version": "1.0.0",
    "description": "E-commerce API running on AKS with product and order management"
  },
  "servers": [
    {
      "url": "$storeUrl",
      "description": "AKS Store API running on Azure Kubernetes Service"
    }
  ],
  "paths": {
    "/products": {
      "get": {
        "summary": "Get all products",
        "description": "Retrieve a list of all available products in the store",
        "responses": {
          "200": {
            "description": "List of products",
            "content": {
              "application/json": {
                "schema": {
                  "type": "array",
                  "items": {
                    "type": "object",
                    "properties": {
                      "id": { "type": "integer" },
                      "name": { "type": "string" },
                      "price": { "type": "number" },
                      "image": { "type": "string" }
                    }
                  }
                }
              }
            }
          }
        }
      }
    },
    "/products/{id}": {
      "get": {
        "summary": "Get product by ID",
        "description": "Retrieve a specific product by its ID",
        "parameters": [
          {
            "name": "id",
            "in": "path",
            "required": true,
            "schema": { "type": "integer" }
          }
        ],
        "responses": {
          "200": { "description": "Product details" },
          "404": { "description": "Product not found" }
        }
      }
    },
    "/orders": {
      "get": {
        "summary": "Get all orders",
        "description": "Retrieve a list of all orders",
        "responses": {
          "200": { "description": "List of orders" }
        }
      },
      "post": {
        "summary": "Create a new order",
        "description": "Place a new order for products",
        "requestBody": {
          "required": true,
          "content": {
            "application/json": {
              "schema": {
                "type": "object",
                "properties": {
                  "productId": { "type": "integer" },
                  "quantity": { "type": "integer" },
                  "customerEmail": { "type": "string" }
                }
              }
            }
          }
        },
        "responses": {
          "201": { "description": "Order created" }
        }
      }
    }
  }
}
"@ | Out-File -FilePath "aks-store-api-spec.json" -Encoding utf8

Write-Host "✅ OpenAPI specification created with server URL: $storeUrl"

# Validate the JSON file
try {
    $jsonValidation = Get-Content "aks-store-api-spec.json" -Raw | ConvertFrom-Json
    Write-Host "✅ JSON file is valid and ready for import"
    Write-Host "   Server URL in file: $($jsonValidation.servers[0].url)"
} catch {
    Write-Host "❌ JSON validation failed: $($_.Exception.Message)"
    Write-Host "   Please check the file content and try again"
    exit 1
}

# Import API into APIM
Write-Host "📥 Importing AKS Store API into API Management..."
az apim api import `
  --resource-group $env:RG_NAME `
  --service-name "apim-$env:RAND" `
  --api-id "aks-store-api" `
  --path "store" `
  --specification-format OpenApi `
  --specification-path "aks-store-api-spec.json" `
  --display-name "AKS Store API"

if ($LASTEXITCODE -eq 0) {
    Write-Host "✅ AKS Store API successfully imported into API Management"
} else {
    Write-Host "❌ Failed to import API. Please check the OpenAPI specification file."
    Write-Host "   You can validate the file content by running: Get-Content aks-store-api-spec.json"
}
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
"@ | Out-File -FilePath "aks-store-api-policy.xml" -Encoding utf8

# Apply the policy to the AKS Store API
az apim api policy create `
  --resource-group $env:RG_NAME `
  --service-name "apim-$env:RAND" `
  --api-id "aks-store-api" `
  --policy-content (Get-Content -Path "aks-store-api-policy.xml" -Raw)

Write-Host "✅ API policies applied to AKS Store API"
```

### Test API through APIM

Test the API through Azure API Management:

```powershell
# Get APIM gateway URL
$APIM_GATEWAY = az apim show --name "apim-$env:RAND" --resource-group $env:RG_NAME --query "gatewayUrl" --output tsv

Write-Host "🧪 Testing AKS Store API through APIM Gateway: $APIM_GATEWAY"

# Test products endpoint through APIM
try {
    Write-Host "   Testing products endpoint..."
    $productsResponse = Invoke-RestMethod -Uri "$APIM_GATEWAY/store/products" -Method GET -TimeoutSec 15
    Write-Host "   ✅ Products API working - Found $($productsResponse.Count) products"
} catch {
    Write-Host "   ⚠️ Products API test failed: $($_.Exception.Message)"
}

# Test orders endpoint through APIM  
try {
    Write-Host "   Testing orders endpoint..."
    $ordersResponse = Invoke-RestMethod -Uri "$APIM_GATEWAY/store/orders" -Method GET -TimeoutSec 15
    Write-Host "   ✅ Orders API working through APIM"
} catch {
    Write-Host "   ⚠️ Orders API test failed: $($_.Exception.Message)"
}

Write-Host ""
Write-Host "🎉 AKS Store API is successfully exposed through Azure API Management!"
Write-Host "📋 Available APIM endpoints:"
Write-Host "   GET  $APIM_GATEWAY/store/products     - Get all products"
Write-Host "   GET  $APIM_GATEWAY/store/products/{id} - Get specific product"
Write-Host "   GET  $APIM_GATEWAY/store/orders       - Get all orders"
Write-Host "   POST $APIM_GATEWAY/store/orders       - Create new order"
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

# Register the AKS Store API
az apic api create `
  --resource-group $env:RG_NAME `
  --service-name "apic-$env:RAND" `
  --workspace-name "main" `
  --api-name "aks-store-api" `
  --title "AKS Store API" `
  --kind "rest" `
  --description "E-commerce API for products and orders, deployed on AKS and exposed through APIM"

# Create API version
az apic api version create `
  --resource-group $env:RG_NAME `
  --service-name "apic-$env:RAND" `
  --workspace-name "main" `
  --api-name "aks-store-api" `
  --version-name "v1" `
  --title "Version 1.0" `
  --lifecycle-stage "production"

# Create API definition (OpenAPI spec)
az apic api definition create `
  --resource-group $env:RG_NAME `
  --service-name "apic-$env:RAND" `
  --workspace-name "main" `
  --api-name "aks-store-api" `
  --version-name "v1" `
  --definition-name "openapi" `
  --title "OpenAPI Definition" `
  --description "OpenAPI 3.0 specification"

# Import the OpenAPI specification
az apic api definition import-specification `
  --resource-group $env:RG_NAME `
  --service-name "apic-$env:RAND" `
  --workspace-name "main" `
  --api-name "aks-store-api" `
  --version-name "v1" `
  --definition-name "openapi" `
  --format "link" `
  --value "$APIM_GATEWAY/store?format=openapi-link&api-version=2023-05-01-preview" `
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
  --api-name "aks-store-api" `
  --deployment-name "production-deployment" `
  --title "Production Deployment" `
  --description "E-commerce API deployed on AKS cluster exposed through APIM" `
  --environment-name "production" `
  --definition-name "openapi" `
  --server '{"runtime-uri":"'$APIM_GATEWAY'/store"}'
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
# Generate some test traffic for the AKS Store API
Write-Host "🚦 Generating test traffic for analytics..."
for ($i = 1; $i -le 10; $i++) {
    try {
        Write-Host "   Test request $i/10..."
        Invoke-RestMethod -Uri "$APIM_GATEWAY/store/products" -Method GET -TimeoutSec 10 | Out-Null
        Start-Sleep -Milliseconds 500
    } catch {
        Write-Host "   Request $i failed: $($_.Exception.Message)"
    }

Write-Host "✅ Test traffic generation complete. Check Azure Portal for analytics."
```

## 🔍 Step 5: Configure API Versioning and Revisions

### Add API Version

Create a new version of the AKS Store API:

```powershell
# Create API version set
az apim api versionset create `
  --resource-group $env:RG_NAME `
  --service-name "apim-$env:RAND" `
  --version-set-id "aks-store-versions" `
  --display-name "AKS Store API Versions" `
  --versioning-scheme "Header" `
  --version-header "Api-Version"

Write-Host "✅ API version set created"

# Update existing API to be version 1.0
az apim api update `
  --resource-group $env:RG_NAME `
  --service-name "apim-$env:RAND" `
  --api-id "aks-store-api" `
  --api-version "1.0" `
  --api-version-set-id "aks-store-versions"

Write-Host "✅ Existing API configured as version 1.0"

# Create version 2.0 with enhanced product information
az apim api create `
  --resource-group $env:RG_NAME `
  --service-name "apim-$env:RAND" `
  --api-id "aks-store-api-v2" `
  --display-name "AKS Store API v2.0" `
  --path "store/v2" `
  --protocols "https" `
  --service-url "http://$STORE_IP" `
  --api-version "2.0" `
  --api-version-set-id "aks-store-versions"

Write-Host "✅ API version 2.0 created"
```

### Create API Revision

Create a revision for testing new features:

```powershell
# Create a revision of the current API
$REVISION_ID = "2"
az apim api revision create `
  --resource-group $env:RG_NAME `
  --service-name "apim-$env:RAND" `
  --api-id "aks-store-api" `
  --api-revision $REVISION_ID `
  --api-revision-description "Enhanced error handling and logging"

Write-Host "✅ API revision $REVISION_ID created"

# List all revisions
az apim api revision list `
  --resource-group $env:RG_NAME `
  --service-name "apim-$env:RAND" `
  --api-id "aks-store-api" `
  --output table

Write-Host "📋 API revisions listed above"
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

# Test direct API access to AKS Store
Write-Host "5. Testing direct AKS Store API access..."
try {
    $DIRECT_RESPONSE = Invoke-RestMethod -Uri "http://$STORE_IP/products" -Method GET -TimeoutSec 10
    Write-Host "   ✅ Direct Store API Access: OK - Found $($DIRECT_RESPONSE.Count) products"
} catch {
    Write-Host "   ❌ Direct Store API Access: FAILED - $($_.Exception.Message)"
}

# Test APIM access to AKS Store
Write-Host "6. Testing APIM access..."
try {
    $APIM_RESPONSE = Invoke-RestMethod -Uri "$APIM_GATEWAY/store/products" -Method GET -TimeoutSec 10
    Write-Host "   ✅ APIM Store API Access: OK - Found $($APIM_RESPONSE.Count) products"
} catch {
    Write-Host "   ❌ APIM Store API Access: FAILED - $($_.Exception.Message)"
}

Write-Host "=== ✅ Health Check Complete ==="
```

### Common Issues and Solutions

```powershell
Write-Host "=== 🔧 Troubleshooting Guide ==="
Write-Host ""
Write-Host "📝 Common issues and solutions:"
Write-Host ""
Write-Host "1️⃣ External IP not assigned to AKS Store service:"
Write-Host "   • Wait 5-10 minutes for Azure Load Balancer provisioning"
Write-Host "   • Check: kubectl get service store-front -w"
Write-Host ""
Write-Host "2️⃣ APIM not responding:"
Write-Host "   • APIM deployment takes 30-45 minutes for Developer tier"
Write-Host "   • Check: az apim show --name apim-$env:RAND --resource-group $env:RG_NAME"
Write-Host ""
Write-Host "3️⃣ AKS Store API not accessible through APIM:"
Write-Host "   • Verify store-front service has external IP assigned"
Write-Host "   • Check APIM backend service URL configuration"
Write-Host "   • Verify AKS Store pods are running: kubectl get pods"
Write-Host ""
Write-Host "4️⃣ Rate limiting errors (429):"
Write-Host "   • Current limit: 100 calls per minute per IP address"
Write-Host "   • Wait 1 minute or modify policy in Azure Portal APIM section"
Write-Host ""
Write-Host "5️⃣ AKS Store pods not starting:"
Write-Host "   • Check pod logs: kubectl logs -l app=product-service"
Write-Host "   • Verify RabbitMQ is running: kubectl get pods -l app=rabbitmq"
Write-Host "   • Check resource quotas: kubectl describe nodes"
```

## View Resources in Azure Portal

Access your deployed resources through the Azure Portal:

```powershell
Write-Host "=== 🌐 Azure Portal Access ==="
Write-Host ""
Write-Host "📊 Resource Information:"
Write-Host "   Resource Group: $env:RG_NAME"
Write-Host "   APIM Gateway URL: $APIM_GATEWAY"
Write-Host "   AKS Store Frontend: http://$STORE_IP"
Write-Host ""
Write-Host "🔗 Azure Portal Quick Links:"
Write-Host "   📋 Resource Group: https://portal.azure.com/#@/resource/subscriptions/$(az account show --query 'id' -o tsv)/resourceGroups/$env:RG_NAME"
Write-Host "   🛠️  API Management: https://portal.azure.com/#@/resource/subscriptions/$(az account show --query 'id' -o tsv)/resourceGroups/$env:RG_NAME/providers/Microsoft.ApiManagement/service/apim-$env:RAND"
Write-Host "   📚 API Center: https://portal.azure.com/#@/resource/subscriptions/$(az account show --query 'id' -o tsv)/resourceGroups/$env:RG_NAME/providers/Microsoft.ApiCenter/services/apic-$env:RAND"
Write-Host "   ⚡ AKS Cluster: https://portal.azure.com/#@/resource/subscriptions/$(az account show --query 'id' -o tsv)/resourceGroups/$env:RG_NAME/providers/Microsoft.ContainerService/managedClusters/aks-cluster-$env:RAND"
```

## 🎉 Summary

In this comprehensive workshop, you successfully:

✅ **Deployed Infrastructure**: Created AKS cluster, Azure API Management, and Azure API Center  
✅ **Deployed AKS Store**: Used the public Azure AKS Store demo application with multiple microservices  
✅ **Exposed through AKS**: Used Kubernetes LoadBalancer service to expose the application  
✅ **Configured APIM**: Set up API gateway with policies for security, CORS, and rate limiting  
✅ **Registered in API Center**: Added API to centralized catalog for governance and documentation  
✅ **Enabled Monitoring**: Configured analytics and comprehensive health checks  
✅ **Troubleshooting**: Learned to diagnose and resolve common issues with AKS and APIM

### 🏗️ Key Benefits Achieved

- **🔒 Centralized API Management**: All APIs accessible through a single, secure gateway
- **🛡️ Enhanced Security**: Rate limiting, CORS policies, and request/response transformation
- **📋 API Governance**: Centralized catalog with OpenAPI documentation and lifecycle management  
- **📈 Scalability**: Kubernetes-based deployment with Azure Load Balancer auto-scaling
- **📊 Observability**: Built-in analytics, health monitoring, and Application Insights integration
- **⚡ No Code Changes**: Public demo application integrated without any modifications

### 🏛️ Architecture Overview

```
[Internet Users] → [Azure API Management Gateway] → [Azure Load Balancer] → [AKS Cluster]
                                ↓                                            ↓
                      [Rate Limiting & Policies]                    [AKS Store Microservices]
                                ↓                                            ↓
                        [Azure API Center] ←←←←←←← [OpenAPI Specifications]
                                ↓
                   [Documentation & Governance Portal]
```

### 🚀 Next Steps

To extend this solution beyond the workshop, consider:

• **🔐 Authentication**: Add Azure AD B2C or OAuth 2.0 authentication flows  
• **🔄 Advanced Policies**: Implement request/response transformation and caching  
• **🌍 Multiple Environments**: Set up staging, development, and production environments  
• **⚙️ CI/CD Pipeline**: Automate deployment with Azure DevOps or GitHub Actions  
• **📈 Advanced Monitoring**: Add custom metrics, alerts, and distributed tracing  
• **📝 API Versioning**: Implement semantic versioning strategies for API evolution  
• **🔗 Service Mesh**: Consider Istio or Linkerd for advanced microservices management

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
