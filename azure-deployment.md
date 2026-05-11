# Complete Azure Container Deployment Workflow

This guide provides the complete, end-to-end script to deploy a local application codebase as a Docker container to Azure App Service (Web Apps) using the Azure Container Registry (ACR).

> **Tip:** Run these commands sequentially in your Azure Cloud Shell or a local terminal where the Azure CLI is installed. Make sure your terminal's working directory is the root folder of your application (where your `Dockerfile` is located).

## 1. Setup Variables
Define unique names for your resources. 
> **Warning:** The `ACR_NAME` and `WEBAPP_NAME` must be globally unique across all of Azure. Change the random numbers at the end before running.

```bash
# Define your resource names
RESOURCE_GROUP="asp-demo-rg"
LOCATION="southeastasia"
ACR_NAME="carvedrockacr0422"                 # Must be globally unique, alphanumeric only
APP_SERVICE_PLAN="asp-carvedrockfitness-linux"
WEBAPP_NAME="carvedrockfitness-container-0422" # Must be globally unique
IMAGE_NAME="fitness-app:v1"
```

## 2. Create the Resource Group
*(Skip this step if your resource group already exists)*
```bash
az group create --name $RESOURCE_GROUP --location $LOCATION
```

## 3. Create the Azure Container Registry (ACR)
Create the private registry to securely store your Docker images.
```bash
az acr create \
  --resource-group $RESOURCE_GROUP \
  --name $ACR_NAME \
  --sku Basic
```

## 4. Build and Push the Docker Image
Build the container image directly in Azure from your local source code and push it to the new registry.
```bash
# Ensure you are in the directory containing your source code and Dockerfile
az acr build --registry $ACR_NAME --image $IMAGE_NAME .
```

## 5. Enable ACR Admin User
Allow Azure Web Apps to authenticate and pull your private image.
```bash
az acr update --name $ACR_NAME --admin-enabled true
```

## 6. Create the Linux App Service Plan
Create the underlying server infrastructure (Linux) to host the container.
```bash
az appservice plan create \
  --name $APP_SERVICE_PLAN \
  --resource-group $RESOURCE_GROUP \
  --is-linux \
  --sku B1
```

## 7. Create the Web App Infrastructure
Create the Web App using a generic public image (`nginx`) to establish the resource cleanly without authentication errors.
```bash
az webapp create \
  --resource-group $RESOURCE_GROUP \
  --plan $APP_SERVICE_PLAN \
  --name $WEBAPP_NAME \
  --deployment-container-image-name nginx
```

## 8. Configure Web App to Use Your Container
Point the newly created Web App to your private image in the ACR. Azure will automatically use the admin credentials we enabled in step 5.
```bash
az webapp config container set \
  --name $WEBAPP_NAME \
  --resource-group $RESOURCE_GROUP \
  --docker-custom-image-name $ACR_NAME.azurecr.io/$IMAGE_NAME \
  --docker-registry-server-url https://$ACR_NAME.azurecr.io
```

## 9. View Your Application
Get the URL of your live application.
```bash
# Print the URL of the newly created web app
echo "Your app is live at: https://$WEBAPP_NAME.azurewebsites.net"
```
> **Note:** The very first time you visit the URL, it may take 1-3 minutes to load as Azure pulls the container image and starts up the application.
