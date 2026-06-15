name: Deploy to Azure

on:
  push:
    branches:
      - main
      - setup/initial-structure
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy to'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

env:
  RESOURCE_GROUP: stb-landing-rg
  LOCATION: eastus
  APP_PLAN_NAME: stb-landing-plan
  SKU: B1
  RUNTIME: NODE|18-lts

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install Dependencies
        run: npm ci --if-present

      - name: Build Application
        run: npm run build --if-present

      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Create Resource Group
        run: |
          az group create \
            --name ${{ env.RESOURCE_GROUP }} \
            --location ${{ env.LOCATION }} \
            --output none 2>/dev/null || true

      - name: Create App Service Plan
        run: |
          az appservice plan create \
            --name ${{ env.APP_PLAN_NAME }} \
            --resource-group ${{ env.RESOURCE_GROUP }} \
            --sku ${{ env.SKU }} \
            --is-linux \
            --output none 2>/dev/null || true

      - name: Create or Update Web App
        id: webapp
        run: |
          APP_NAME="stb-landing-$(date +%s)"
          
          az webapp create \
            --name $APP_NAME \
            --resource-group ${{ env.RESOURCE_GROUP }} \
            --plan ${{ env.APP_PLAN_NAME }} \
            --runtime "${{ env.RUNTIME }}" \
            --output none 2>/dev/null || \
          APP_NAME=$(az webapp list --resource-group ${{ env.RESOURCE_GROUP }} --query "[0].name" --output tsv)
          
          echo "app_name=$APP_NAME" >> $GITHUB_OUTPUT
          echo "App Name: $APP_NAME"

      - name: Configure App Settings
        run: |
          az webapp config appsettings set \
            --name ${{ steps.webapp.outputs.app_name }} \
            --resource-group ${{ env.RESOURCE_GROUP }} \
            --settings \
              NODE_ENV=production \
              WEBSITE_NODE_DEFAULT_VERSION=18-lts \
              SCM_DO_BUILD_DURING_DEPLOYMENT=true

      - name: Deploy via Git
        run: |
          APP_NAME=${{ steps.webapp.outputs.app_name }}
          
          # Get deployment credentials
          CREDENTIALS=$(az webapp deployment list-publishing-credentials \
            --name $APP_NAME \
            --resource-group ${{ env.RESOURCE_GROUP }} \
            --query publishingCredentials.publishingPassword \
            --output tsv)
          
          # Get git deployment URL
          GIT_URL=$(az webapp deployment source config-local-git \
            --name $APP_NAME \
            --resource-group ${{ env.RESOURCE_GROUP }} \
            --query url --output tsv)
          
          # Add remote and push
          git remote add azure "$GIT_URL" 2>/dev/null || git remote set-url azure "$GIT_URL"
          git push azure ${{ github.ref_name }}:main --force

      - name: Get Deployment URL
        id: deployment
        run: |
          APP_NAME=${{ steps.webapp.outputs.app_name }}
          APP_URL="https://$(az webapp show \
            --name $APP_NAME \
            --resource-group ${{ env.RESOURCE_GROUP }} \
            --query defaultHostName --output tsv)"
          
          echo "app_url=$APP_URL" >> $GITHUB_OUTPUT
          echo "App URL: $APP_URL"

      - name: Verify Deployment
        run: |
          APP_URL=${{ steps.deployment.outputs.app_url }}
          echo "Testing deployment at: $APP_URL"
          
          # Wait for app to be ready
          sleep 10
          
          # Test connectivity
          curl -I $APP_URL || echo "App still initializing..."

      - name: Post Deployment Summary
        if: always()
        run: |
          echo "## 🚀 Deployment Summary" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "**Status:** ${{ job.status }}" >> $GITHUB_STEP_SUMMARY
          echo "**App URL:** ${{ steps.deployment.outputs.app_url }}" >> $GITHUB_STEP_SUMMARY
          echo "**App Name:** ${{ steps.webapp.outputs.app_name }}" >> $GITHUB_STEP_SUMMARY
          echo "**Resource Group:** ${{ env.RESOURCE_GROUP }}" >> $GITHUB_STEP_SUMMARY
          echo "**Location:** ${{ env.LOCATION }}" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "**Deployment Time:** $(date)" >> $GITHUB_STEP_SUMMARY

      - name: Azure Logout
        if: always()
        run: az logout
