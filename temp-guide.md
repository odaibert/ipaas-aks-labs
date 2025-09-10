---
title: "Simplifying API Management on AKS with Azure API Management & API Center"
sidebar_label: "API Management on AKS"
sidebar_position: 1
---

# Simplifying API Management on AKS with Azure API Management & API Center

## Objective

This comprehensive guide will show you how to deploy a modern API management solution using Azure Kubernetes Service (AKS), Azure API Management (APIM), and Azure API Center. You'll learn to expose, secure, document, and govern APIs running on AKS without complex dependencies or external tools.

After completing this workshop, you will be able to:

• Deploy a sample API to an AKS cluster
• Set up Azure API Management to expose and secure your APIs
• Register APIs in Azure API Center for discovery and governance
• Configure authentication, rate limiting, and monitoring
• Troubleshoot common API management issues
• Clean up all resources when finished

> ⏱️ **Estimated time to complete**: 60-75 minutes

## Prerequisites

Before you begin, you will need an [Azure subscription](https://azure.microsoft.com) with Contributor permissions and basic familiarity with Azure Portal.

In addition, you will need the following tools installed on your local machine:

• [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) (version 2.0.80 or later)
• PowerShell (available on Windows, macOS, and Linux)

:::info
This workshop uses PowerShell commands that work across Windows, macOS, and Linux. If you prefer, you can adapt the commands to work with Bash on Linux/macOS systems.
:::

### Setup Azure CLI

Start by logging into Azure by running the following command and follow the prompts:

```bash
az login --use-device-code
```

:::tip
You can log into a different tenant by passing in the `--tenant` flag to specify your tenant domain or tenant ID.
:::

Register required resource providers for this workshop:

```bash
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.ApiManagement
az provider register --namespace Microsoft.ApiCenter
```

Check the status of the resource provider registration:

```bash
az provider show --namespace Microsoft.ContainerService --query "registrationState"
az provider show --namespace Microsoft.ApiManagement --query "registrationState"
az provider show --namespace Microsoft.ApiCenter --query "registrationState"
```

:::warning
Make sure all providers show "Registered" status before proceeding.
:::

### Setup Resource Group

In this workshop, we will set environment variables for the resource group name and location.

:::important
The following commands will set the environment variables for your current terminal session. If you close the terminal, you will need to set the variables again.
:::

To keep resource names unique, we will use a random number as a suffix:

```bash
RAND=$RANDOM
export RAND
echo "Random resource identifier will be: ${RAND}"
```

Set the location to a region of your choice. For example, `eastus` or `westeurope`:

```bash
export LOCATION=eastus
```

Create a resource group name using the random number:

```bash
export RG_NAME=apim-aks-lab-$RAND
echo "Resource group and location configured:"
echo "  Resource Group: ${RG_NAME}"
echo "  Location: ${LOCATION}"
```

:::tip
You can list available regions with:
```bash
az account list-locations --query "[].{Region:name}" --output table
```
:::

Run the following command to create a resource group:

```bash
az group create --name ${RG_NAME} --location ${LOCATION}
```

## Deploy Infrastructure Components

### Create AKS Cluster

First, let's create an AKS cluster to host our sample API:

```bash
az aks create \
  --resource-group ${RG_NAME} \
  --name aks-cluster-${RAND} \
  --node-count 2 \
  --node-vm-size Standard_B2s \
  --enable-managed-identity \
  --generate-ssh-keys \
  --enable-addons monitoring
```

:::info
This will take 5-10 minutes to complete. The cluster will be created with monitoring enabled and managed identity for secure Azure service integration.
:::

Get credentials for the AKS cluster:

```bash
az aks get-credentials --resource-group ${RG_NAME} --name aks-cluster-${RAND}
```

### Create Azure API Management Instance

Create an API Management instance to act as our API gateway:

```bash
az apim create \
  --resource-group ${RG_NAME} \
  --name apim-${RAND} \
  --publisher-email "admin@contoso.com" \
  --publisher-name "Contoso API Management" \
  --sku-name Developer
```

:::warning
This process can take 30-45 minutes to complete. Do not close the terminal while this is running.
:::

### Create Azure API Center

Create an API Center for API discovery and governance:

```bash
az apic create \
  --resource-group ${RG_NAME} \
  --name apic-${RAND} \
  --location ${LOCATION}
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

:::info
This application is perfect for our demo because:
- **Pre-built APIs**: Multiple REST endpoints available for testing
- **Real-world architecture**: Demonstrates microservices patterns
:::
> - **No build required**: Uses public container images
> - **Production-ready**: Includes health checks and proper service design

#### Step 1: Create Application Namespace

First, let's create a dedicated namespace for our demo application:

```bash
# Create namespace for the AKS Store demo
kubectl create namespace aks-store-demo
echo "✅ Created aks-store-demo namespace"
```

#### Step 2: Deploy the Application

Deploy the complete AKS Store application using the pre-built manifest:

```bash
# Deploy the AKS Store demo application
echo "🚀 Deploying AKS Store Demo application..."
echo "   This includes: store-front, product-service, order-service, and rabbitmq"

kubectl apply -n aks-store-demo -f https://raw.githubusercontent.com/Azure-Samples/aks-store-demo/main/aks-store-ingress-quickstart.yaml

echo "✅ AKS Store Demo application deployed successfully!"
```

#### Step 3: Verify Deployment Status

Check that all components are running correctly:

```bash
# Wait for pods to be ready
echo "⏳ Waiting for pods to start (this may take 2-3 minutes)..."
sleep 30

# Check pod status
echo "📋 Checking pod status:"
kubectl get pods -n aks-store-demo

echo ""
echo "🔍 Checking services:"
kubectl get services -n aks-store-demo

echo ""
echo "🌐 Checking ingress status:"
kubectl get ingress -n aks-store-demo
```

#### Step 4: Get Application External IP

Expose the store-front service and wait for external IP assignment:

```bash
# Expose the store-front service as LoadBalancer to get an external IP
echo "🔗 Exposing store-front service with LoadBalancer..."
kubectl patch service store-front -n aks-store-demo -p '{"spec":{"type":"LoadBalancer"}}'

# Wait for external IP assignment
echo "⏳ Waiting for external IP assignment..."
timeout=300  # 5 minutes timeout
elapsed=0
while true; do
    EXTERNAL_IP=$(kubectl get service store-front -n aks-store-demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
    if [[ -z "$EXTERNAL_IP" ]]; then
        echo "   Still waiting for IP assignment... ($elapsed seconds elapsed)"
        sleep 15
        elapsed=$((elapsed + 15))
        if [[ $elapsed -ge $timeout ]]; then
            echo "   ⚠️ Timeout reached. This may take up to 10-15 minutes in some regions."
            echo "   ℹ️ You can continue and check later with: kubectl get service store-front -n aks-store-demo"
            break
        fi
    else
        break
    fi
done

if [[ -n "$EXTERNAL_IP" ]]; then
    echo "✅ External IP assigned: $EXTERNAL_IP"
    export STORE_IP=$EXTERNAL_IP
else
    echo "⏸️ External IP assignment in progress. Please run this command later:"
    echo "   export STORE_IP=\$(kubectl get service store-front -n aks-store-demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
fi
```

#### Step 5: Test the Application

Test the application and its API endpoints:

```bash
# Test the store front application
echo "🧪 Testing the AKS Store application..."

# Check if we have the external IP
if ([string]::IsNullOrEmpty($env:STORE_IP)) {
    echo "⚠️ External IP not yet available. Getting current status..."
    $env:STORE_IP = kubectl get service store-front -n aks-store-demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
}

if (-not [string]::IsNullOrEmpty($env:STORE_IP)) {
    # Test the main store front
    try {
        echo "   Testing store front web app..."
        $response = Invoke-WebRequest -Uri "http://$env:STORE_IP" -UseBasicParsing -TimeoutSec 10
        echo "   ✅ Store front is accessible (Status: $($response.StatusCode))"
    } catch {
        echo "   ⚠️ Store front not yet ready: $($_.Exception.Message)"
    }

    # Test the product service API
    try {
        echo "   Testing product service API..."
        $productResponse = Invoke-RestMethod -Uri "http://$env:STORE_IP/products" -TimeoutSec 10
        echo "   ✅ Product API is working - Found $($productResponse.Count) products"
    } catch {
        echo "   ⚠️ Product API not yet ready: $($_.Exception.Message)"
    }

    echo ""
    echo "🎯 Available API Endpoints for APIM Integration:"
    echo "   GET  http://$env:STORE_IP/products        - Get all products"
    echo "   GET  http://$env:STORE_IP/products/{id}   - Get specific product"
    echo "   POST http://$env:STORE_IP/orders          - Create new order"
    echo "   GET  http://$env:STORE_IP/orders          - Get all orders"
} else {
    echo "❌ External IP not yet assigned. Please wait a few more minutes and try again."
    echo "ℹ️ Check status with: kubectl get service store-front -n aks-store-demo"
}
echo ""
echo "📊 Application Features:"
echo "   - Multi-service architecture with microservices"
echo "   - REST API endpoints for products and orders"
echo "   - Web frontend for customer interaction"
echo "   - Message queue integration with RabbitMQ"
echo "   - Production-ready with health checks"
```

:::tip
You can view the AKS Store application in your browser by navigating to `http://$env:STORE_IP` once the external IP is assigned. The application provides a complete e-commerce experience with product catalog and order functionality.
:::

### Test the Deployed Application

Our AKS Store Demo application is now running and ready to be integrated with Azure API Management. Let's verify everything is working:

```powershell
# Verify all services are running
echo "🔍 Final verification of deployed services:"
kubectl get all -n aks-store-demo

echo ""
echo "📱 Application Access:"
echo "   Store Front Web App: http://$env:STORE_IP"
echo "   Products API: http://$env:STORE_IP/products"
echo "   Orders API: http://$env:STORE_IP/orders"
echo ""
echo "✅ AKS Store Demo is ready for API Management integration!"
```

## 🔗 Step 6: Configure Azure API Management

### Import API into APIM

Now let's expose our AKS Store APIs through Azure API Management.

:::info Important Note
This step will automatically generate an OpenAPI specification file (`aks-store-api-spec.json`) that contains your environment-specific IP addresses. This file is not tracked in git to ensure it always reflects your current deployment.
:::

```powershell
# Ensure STORE_IP is set
if ([string]::IsNullOrEmpty($env:STORE_IP)) {
    echo "⚠️ STORE_IP not set. Getting external IP..."
    $env:STORE_IP = kubectl get service store-front -n aks-store-demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
    
    if ([string]::IsNullOrEmpty($env:STORE_IP)) {
        echo "❌ External IP not yet available. Please wait for LoadBalancer to assign IP and run:"
        echo "   `$env:STORE_IP = kubectl get service store-front -n aks-store-demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}'"
        exit 1
    }
}

echo "✅ Using AKS Store IP: $env:STORE_IP"

# Get APIM service URL
$APIM_URL = az apim show --name "apim-$env:RAND" --resource-group $env:RG_NAME --query "gatewayUrl" --output tsv

# Create OpenAPI specification for the AKS Store APIs
# Method: Create JSON content with proper variable substitution
echo "📝 Creating OpenAPI specification..."

$storeUrl = "http://$env:STORE_IP"
$jsonContent = @"
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
"@

# Save the JSON content to file
$jsonContent | Out-File -FilePath "aks-store-api-spec.json" -Encoding utf8

echo "✅ OpenAPI specification created with server URL: $storeUrl"

# Validate the JSON file
try {
    $jsonValidation = Get-Content "aks-store-api-spec.json" -Raw | ConvertFrom-Json
    echo "✅ JSON file is valid and ready for import"
    echo "   Server URL in file: $($jsonValidation.servers[0].url)"
    echo "   File created: aks-store-api-spec.json"
} catch {
    echo "❌ JSON validation failed: $($_.Exception.Message)"
    echo "   Please check the file content and try again"
    exit 1
}

echo ""
echo "ℹ️  Note: The JSON specification file is auto-generated and not tracked in git"
echo "   This ensures the file always contains your current environment's IP addresses"

# Import API into APIM
echo "📥 Importing AKS Store API into API Management..."
az apim api import `
  --resource-group $env:RG_NAME `
  --service-name "apim-$env:RAND" `
  --api-id "aks-store-api" `
  --path "store" `
  --specification-format OpenApi `
  --specification-path "aks-store-api-spec.json" `
  --display-name "AKS Store API"

if ($LASTEXITCODE -eq 0) {
    echo "✅ AKS Store API successfully imported into API Management"
} else {
    echo "❌ Failed to import API. Please check the OpenAPI specification file."
    echo "   You can validate the file content by running: Get-Content aks-store-api-spec.json"
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

echo "✅ API policies applied to AKS Store API"
```

### Test API through APIM

Test the API through Azure API Management:

```powershell
# Get APIM gateway URL
$APIM_GATEWAY = az apim show --name "apim-$env:RAND" --resource-group $env:RG_NAME --query "gatewayUrl" --output tsv

echo "🧪 Testing AKS Store API through APIM Gateway: $APIM_GATEWAY"

# Test products endpoint through APIM
try {
    echo "   Testing products endpoint..."
    $productsResponse = Invoke-RestMethod -Uri "$APIM_GATEWAY/store/products" -Method GET -TimeoutSec 15
    echo "   ✅ Products API working - Found $($productsResponse.Count) products"
} catch {
    echo "   ⚠️ Products API test failed: $($_.Exception.Message)"
}

# Test orders endpoint through APIM  
try {
    echo "   Testing orders endpoint..."
    $ordersResponse = Invoke-RestMethod -Uri "$APIM_GATEWAY/store/orders" -Method GET -TimeoutSec 15
    echo "   ✅ Orders API working through APIM"
} catch {
    echo "   ⚠️ Orders API test failed: $($_.Exception.Message)"
}

echo ""
echo "🎉 AKS Store API is successfully exposed through Azure API Management!"
echo "📋 Available APIM endpoints:"
echo "   GET  $APIM_GATEWAY/store/products     - Get all products"
echo "   GET  $APIM_GATEWAY/store/products/{id} - Get specific product"
echo "   GET  $APIM_GATEWAY/store/orders       - Get all orders"
echo "   POST $APIM_GATEWAY/store/orders       - Create new order"
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

echo "Application Insights configured with key: $INSTRUMENTATION_KEY"
```

### View API Analytics

Check API usage and performance:

```powershell
# Generate some test traffic for the AKS Store API
echo "🚦 Generating test traffic for analytics..."
for ($i = 1; $i -le 10; $i++) {
    try {
        echo "   Test request $i/10..."
        Invoke-RestMethod -Uri "$APIM_GATEWAY/store/products" -Method GET -TimeoutSec 10 | Out-Null
        Start-Sleep -Milliseconds 500
    } catch {
        echo "   Request $i failed: $($_.Exception.Message)"
    }

echo "✅ Test traffic generation complete. Check Azure Portal for analytics."
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

echo "✅ API version set created"

# Update existing API to be version 1.0
az apim api update `
  --resource-group $env:RG_NAME `
  --service-name "apim-$env:RAND" `
  --api-id "aks-store-api" `
  --api-version "1.0" `
  --api-version-set-id "aks-store-versions"

echo "✅ Existing API configured as version 1.0"

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

echo "✅ API version 2.0 created"
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

echo "✅ API revision $REVISION_ID created"

# List all revisions
az apim api revision list `
  --resource-group $env:RG_NAME `
  --service-name "apim-$env:RAND" `
  --api-id "aks-store-api" `
  --output table

echo "📋 API revisions listed above"
```

## Troubleshooting Common Issues

### Check API Health

Verify that all components are working correctly:

```powershell
echo "=== System Health Check ==="

# Check AKS cluster status
echo "1. Checking AKS cluster..."
$AKS_STATUS = az aks show --name "aks-cluster-$env:RAND" --resource-group $env:RG_NAME --query "provisioningState" --output tsv
echo "   AKS Cluster Status: $AKS_STATUS"

# Check pods status
echo "2. Checking pod status..."
kubectl get pods -o wide

# Check services
echo "3. Checking services..."
kubectl get services

# Check APIM status
echo "4. Checking APIM status..."
$APIM_STATUS = az apim show --name "apim-$env:RAND" --resource-group $env:RG_NAME --query "provisioningState" --output tsv
echo "   APIM Status: $APIM_STATUS"

# Test direct API access to AKS Store
echo "5. Testing direct AKS Store API access..."
try {
    $DIRECT_RESPONSE = Invoke-RestMethod -Uri "http://$STORE_IP/products" -Method GET -TimeoutSec 10
    echo "   ✅ Direct Store API Access: OK - Found $($DIRECT_RESPONSE.Count) products"
} catch {
    echo "   ❌ Direct Store API Access: FAILED - $($_.Exception.Message)"
}

# Test APIM access to AKS Store
echo "6. Testing APIM access..."
try {
    $APIM_RESPONSE = Invoke-RestMethod -Uri "$APIM_GATEWAY/store/products" -Method GET -TimeoutSec 10
    echo "   ✅ APIM Store API Access: OK - Found $($APIM_RESPONSE.Count) products"
} catch {
    echo "   ❌ APIM Store API Access: FAILED - $($_.Exception.Message)"
}

echo "=== ✅ Health Check Complete ==="
```

### Common Issues and Solutions

```powershell
echo "=== 🔧 Troubleshooting Guide ==="
echo ""
echo "📝 Common issues and solutions:"
echo ""
echo "1️⃣ External IP not assigned to AKS Store service:"
echo "   • Wait 5-10 minutes for Azure Load Balancer provisioning"
echo "   • Check: kubectl get service store-front -w"
echo ""
echo "2️⃣ APIM not responding:"
echo "   • APIM deployment takes 30-45 minutes for Developer tier"
echo "   • Check: az apim show --name apim-$env:RAND --resource-group $env:RG_NAME"
echo ""
echo "3️⃣ AKS Store API not accessible through APIM:"
echo "   • Verify store-front service has external IP assigned"
echo "   • Check APIM backend service URL configuration"
echo "   • Verify AKS Store pods are running: kubectl get pods"
echo ""
echo "4️⃣ Rate limiting errors (429):"
echo "   • Current limit: 100 calls per minute per IP address"
echo "   • Wait 1 minute or modify policy in Azure Portal APIM section"
echo ""
echo "5️⃣ AKS Store pods not starting:"
echo "   • Check pod logs: kubectl logs -l app=product-service"
echo "   • Verify RabbitMQ is running: kubectl get pods -l app=rabbitmq"
echo "   • Check resource quotas: kubectl describe nodes"
```

## View Resources in Azure Portal

Access your deployed resources through the Azure Portal:

```powershell
echo "=== 🌐 Azure Portal Access ==="
echo ""
echo "📊 Resource Information:"
echo "   Resource Group: $env:RG_NAME"
echo "   APIM Gateway URL: $APIM_GATEWAY"
echo "   AKS Store Frontend: http://$STORE_IP"
echo ""
echo "🔗 Azure Portal Quick Links:"
echo "   📋 Resource Group: https://portal.azure.com/#@/resource/subscriptions/$(az account show --query 'id' -o tsv)/resourceGroups/$env:RG_NAME"
echo "   🛠️  API Management: https://portal.azure.com/#@/resource/subscriptions/$(az account show --query 'id' -o tsv)/resourceGroups/$env:RG_NAME/providers/Microsoft.ApiManagement/service/apim-$env:RAND"
echo "   📚 API Center: https://portal.azure.com/#@/resource/subscriptions/$(az account show --query 'id' -o tsv)/resourceGroups/$env:RG_NAME/providers/Microsoft.ApiCenter/services/apic-$env:RAND"
echo "   ⚡ AKS Cluster: https://portal.azure.com/#@/resource/subscriptions/$(az account show --query 'id' -o tsv)/resourceGroups/$env:RG_NAME/providers/Microsoft.ContainerService/managedClusters/aks-cluster-$env:RAND"
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

:::warning
This will permanently delete all resources created in this workshop. Make sure you no longer need these resources.
:::

To clean up all resources created in this workshop:

```powershell
echo "Starting cleanup process..."

# Delete the resource group and all its contents
az group delete `
  --name $env:RG_NAME `
  --yes `
  --no-wait

echo "Cleanup initiated. Resources are being deleted in the background."
echo "This process may take 10-15 minutes to complete."
echo ""
echo "You can monitor the deletion progress in the Azure Portal:"
echo "https://portal.azure.com/#blade/HubsExtension/BrowseResourceGroups"
```

### Cleanup Verification

To verify cleanup completion:

```powershell
# Check if resource group still exists
$RG_EXISTS = az group exists --name $env:RG_NAME
if ($RG_EXISTS -eq "true") {
    echo "Resource group still exists. Cleanup in progress..."
} else {
    echo "✅ Cleanup complete! All resources have been deleted."
}

# Clear environment variables
Remove-Variable -Name "RG_NAME" -ErrorAction SilentlyContinue
Remove-Variable -Name "LOCATION" -ErrorAction SilentlyContinue
Remove-Variable -Name "RAND" -ErrorAction SilentlyContinue
Remove-Variable -Name "API_EXTERNAL_IP" -ErrorAction SilentlyContinue

echo "Environment variables cleared."
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
