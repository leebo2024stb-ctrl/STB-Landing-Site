# Azure Deployment Guide: 30-Minute Quick Start

Deploy your STB Landing Site to Azure in 30 minutes using this guided approach.

## Prerequisites (5 minutes)

### Required Tools
- **Azure CLI** - [Install here](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
- **Git** - [Install here](https://git-scm.com/downloads)
- **Node.js 18+** (if needed) - [Install here](https://nodejs.org/)

### Azure Account Setup
- Active Azure subscription ([Free trial](https://azure.microsoft.com/en-us/free/))
- Owner or Contributor access

### Verify Installation
```bash
az --version
git --version
```

---

## Phase 1: Authentication Setup (3 minutes)

### Step 1: Login to Azure
```bash
az login
```
This opens your browser to authenticate. Return to the terminal once complete.

### Step 2: Set Default Subscription
```bash
# List available subscriptions
az account list --output table

# Set the default (replace with your subscription ID)
az account set --subscription "YOUR_SUBSCRIPTION_ID"

# Verify
az account show
```

---

## Phase 2: Resource Group Creation (2 minutes)

### Step 3: Create Resource Group
```bash
# Variables
RESOURCE_GROUP="stb-landing-rg"
LOCATION="eastus"  # Change to your preferred region

# Create resource group
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION

# Verify
az group show --name $RESOURCE_GROUP
```

**Recommended Regions:** `eastus`, `westus2`, `northeurope`, `southeastasia`

---

## Phase 3: App Service Setup (10 minutes)

### Step 4: Create App Service Plan
```bash
# Variables
APP_PLAN_NAME="stb-landing-plan"
SKU="B1"  # Basic tier (sufficient for landing pages)

# Create plan
az appservice plan create \
  --name $APP_PLAN_NAME \
  --resource-group $RESOURCE_GROUP \
  --sku $SKU \
  --is-linux

# Verify
az appservice plan show \
  --name $APP_PLAN_NAME \
  --resource-group $RESOURCE_GROUP
```

**SKU Options:**
- `B1` - Basic (shared resources, ~$11/month)
- `S1` - Standard (dedicated, ~$74/month)
- `P1V2` - Premium (high performance)

### Step 5: Create Web App
```bash
# Variables
APP_NAME="stb-landing-$(date +%s)"  # Unique name (timestamp)

# Create web app
az webapp create \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --plan $APP_PLAN_NAME \
  --runtime "NODE|18-lts"

echo "App Name: $APP_NAME"

# Enable git deployment
az webapp deployment source config-local-git \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP

# Get deployment URL
GIT_URL=$(az webapp deployment source config-local-git \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query url --output tsv)

echo "Git Remote URL: $GIT_URL"
```

### Step 6: Configure Deployment Credentials
```bash
# Create deployment user (use simple username/password)
az webapp deployment user set \
  --user-name "stbdeployuser" \
  --password "P@ssw0rd123!"  # Use a strong password!

# Or retrieve existing credentials
az webapp deployment list-publishing-credentials \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "[].publishingUserName"
```

---

## Phase 4: Git Integration (5 minutes)

### Step 7: Configure Local Git Remote
```bash
# Navigate to your repository
cd ~/path/to/STB-Landing-Site

# Add Azure as remote
git remote add azure "$GIT_URL"

# Verify
git remote -v
```

### Step 8: Deploy via Git Push
```bash
# Push to Azure (will trigger deployment)
git push azure setup/initial-structure:main

# Monitor logs
az webapp log tail \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP
```

### Step 9: Verify Deployment
```bash
# Get app URL
APP_URL=$(az webapp show \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query defaultHostName --output tsv)

echo "Your app is live at: https://$APP_URL"

# Test with curl
curl -I https://$APP_URL
```

---

## Phase 5: Configuration & Optimization (5 minutes)

### Step 10: Set Environment Variables
```bash
# Add app settings
az webapp config appsettings set \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --settings \
    NODE_ENV=production \
    WEBSITE_NODE_DEFAULT_VERSION=18-lts

# View settings
az webapp config appsettings list \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP
```

### Step 11: Enable Logging
```bash
# Enable application logging
az webapp log config \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --docker-container-logging filesystem \
  --level Verbose

# Stream logs in real-time
az webapp log tail \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --follow
```

### Step 12: Custom Domain (Optional)
```bash
# Add custom domain
az webapp config hostname add \
  --webapp-name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --hostname "yourdomain.com"

# Verify DNS records needed
az webapp config hostname list \
  --webapp-name $APP_NAME \
  --resource-group $RESOURCE_GROUP
```

---

## Automated Deployment Script

Save this as `deploy-to-azure.sh`:

```bash
#!/bin/bash
set -e

# Color codes
GREEN='\033[0;32m'
BLUE='\033[0;34m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo -e "${BLUE}=== STB Landing Site - Azure Deployment ===${NC}\n"

# Phase 1: Verify prerequisites
echo -e "${YELLOW}[1/5] Verifying prerequisites...${NC}"
command -v az &> /dev/null || { echo "Azure CLI not found. Install: https://learn.microsoft.com/en-us/cli/azure/install-azure-cli"; exit 1; }
command -v git &> /dev/null || { echo "Git not found. Install: https://git-scm.com/downloads"; exit 1; }
echo -e "${GREEN}✓ Prerequisites verified${NC}\n"

# Phase 2: Authentication
echo -e "${YELLOW}[2/5] Setting up Azure authentication...${NC}"
az account show &> /dev/null || az login
SUBSCRIPTION=$(az account show --query id --output tsv)
echo -e "${GREEN}✓ Using subscription: $SUBSCRIPTION${NC}\n"

# Phase 3: Create resources
echo -e "${YELLOW}[3/5] Creating Azure resources...${NC}"
RESOURCE_GROUP="stb-landing-rg"
LOCATION="eastus"
APP_PLAN_NAME="stb-landing-plan"
APP_NAME="stb-landing-$(date +%s)"

az group create --name $RESOURCE_GROUP --location $LOCATION --output none
echo -e "${GREEN}✓ Resource group created${NC}"

az appservice plan create --name $APP_PLAN_NAME --resource-group $RESOURCE_GROUP --sku B1 --is-linux --output none
echo -e "${GREEN}✓ App Service plan created${NC}"

az webapp create --name $APP_NAME --resource-group $RESOURCE_GROUP --plan $APP_PLAN_NAME --runtime "NODE|18-lts" --output none
echo -e "${GREEN}✓ Web app created: $APP_NAME${NC}\n"

# Phase 4: Git deployment
echo -e "${YELLOW}[4/5] Configuring Git deployment...${NC}"
az webapp deployment source config-local-git --name $APP_NAME --resource-group $RESOURCE_GROUP --output none
GIT_URL=$(az webapp deployment source config-local-git --name $APP_NAME --resource-group $RESOURCE_GROUP --query url --output tsv)
echo -e "${GREEN}✓ Git remote: $GIT_URL${NC}\n"

# Phase 5: Deploy
echo -e "${YELLOW}[5/5] Deploying application...${NC}"
git remote add azure "$GIT_URL" 2>/dev/null || git remote set-url azure "$GIT_URL"
git push azure $(git branch --show-current):main
echo -e "${GREEN}✓ Deployment started${NC}\n"

# Get app URL
APP_URL="https://$(az webapp show --name $APP_NAME --resource-group $RESOURCE_GROUP --query defaultHostName --output tsv)"
echo -e "${GREEN}=== Deployment Complete ===${NC}"
echo -e "App URL: ${BLUE}$APP_URL${NC}"
echo -e "Resource Group: ${BLUE}$RESOURCE_GROUP${NC}"
echo -e "App Name: ${BLUE}$APP_NAME${NC}"
```

**Usage:**
```bash
chmod +x deploy-to-azure.sh
./deploy-to-azure.sh
```

---

## Troubleshooting

### Deployment Failed
```bash
# View detailed logs
az webapp log tail --name $APP_NAME --resource-group $RESOURCE_GROUP --follow

# Check deployment history
az webapp deployment list --name $APP_NAME --resource-group $RESOURCE_GROUP
```

### Git Push Authentication Error
```bash
# Reconfigure credentials
az webapp deployment user set --user-name "stbdeployuser" --password "NewPassword123!"

# Retry push with authentication prompt
git push azure setup/initial-structure:main
```

### App Not Starting
```bash
# Verify app settings
az webapp config appsettings list --name $APP_NAME --resource-group $RESOURCE_GROUP

# Check Node.js version
az webapp list-runtimes --os linux | grep NODE

# Restart app
az webapp restart --name $APP_NAME --resource-group $RESOURCE_GROUP
```

---

## Cost Management

### Estimate Monthly Cost
- **App Service Plan (B1):** ~$11/month
- **Storage:** ~$1-5/month
- **Data transfer (first 1GB free):** Variable

### Scale Down to Free Tier (Development)
```bash
az appservice plan update \
  --name $APP_PLAN_NAME \
  --resource-group $RESOURCE_GROUP \
  --sku FREE
```

### Delete Resources When Done
```bash
az group delete \
  --name $RESOURCE_GROUP \
  --yes --no-wait
```

---

## Continuous Deployment (CD)

### Enable Auto-Deploy on Git Push
```bash
az webapp deployment source config \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --repo-url "https://github.com/leebo2024stb-ctrl/STB-Landing-Site" \
  --branch "setup/initial-structure" \
  --manual-integration
```

### GitHub Actions Alternative
See `GITHUB_ACTIONS_AZURE.md` for automated CI/CD setup.

---

## Next Steps

1. ✅ **Verify app is running** - Visit the app URL
2. ✅ **Configure custom domain** - Add your domain
3. ✅ **Enable SSL/TLS** - Use Azure-managed certificates
4. ✅ **Set up monitoring** - Enable Application Insights
5. ✅ **Configure backup** - Enable daily backups

---

## Support & Resources

- **Azure Documentation:** https://learn.microsoft.com/en-us/azure/
- **App Service Docs:** https://learn.microsoft.com/en-us/azure/app-service/
- **Azure CLI Reference:** https://learn.microsoft.com/en-us/cli/azure/reference-index
- **Pricing Calculator:** https://azure.microsoft.com/en-us/pricing/calculator/

---

**Time Breakdown:**
- Prerequisites: 5 min
- Authentication: 3 min
- Resource Creation: 2 min
- App Service Setup: 10 min
- Git Integration: 5 min
- Configuration: 5 min
- **Total: ~30 minutes**
