# Tutorial: Deploy environments in CI/CD with GitHub

This template repository contains starter files and configuration for the [Tutorial: Deploy environments in CI/CD with GitHub](https://learn.microsoft.com/en-us/azure/deployment-environments/tutorial-deploy-environments-in-cicd-github).

In this tutorial, you'll Learn how to integrate Azure Deployment Environments into your CI/CD pipeline by using GitHub Actions. You use a workflow that features three branches: main, dev, and test.

![Screenshot showing GitHub Actions workflows](/media/samples-repo-image.png)

## Features

In this tutorial, you learn how to:
1. Configure deployment identities
1. Configure GitHub environments
1. Test the CI/CD pipeline

## Getting Started

### Prereqs
- Dev center
- Catalog and Catalog items in Dev Center
- Dev center Project
- PAT

### CICD Getting started

- Create Repo from this template
- Open codespace

```bash
az login --use-device-code
```

```bash
# Set subscription if more than 1, or not set. Login with owner to sub
az account set -s "Microsoft Azure Sponsorship"
```

```bash
# Set Env Variables for Github
AZURE_DEVCENTER=devcenter-sample
AZURE_PROJECT=project-sample
AZURE_CATALOG=catalog-sample
AZURE_CATALOG_ITEM=FunctionApp
AZURE_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
AZURE_TENANT_ID=$(az account show --query tenantId -o tsv)
GITHUB_TOKEN=< PAT >

gh variable set AZURE_DEVCENTER --body "${AZURE_DEVCENTER}"
gh variable set AZURE_PROJECT --body "${AZURE_PROJECT}" 
gh variable set AZURE_CATALOG --body "${AZURE_CATALOG}" 
gh variable set AZURE_CATALOG_ITEM --body "${AZURE_CATALOG_ITEM}" 
gh variable set AZURE_SUBSCRIPTION_ID --body "${AZURE_SUBSCRIPTION_ID}" 
gh variable set AZURE_TENANT_ID --body "${AZURE_TENANT_ID}" 
```

```bash
# Create the App and Federated Identity
APP_DISPLAY_NAME="$AZURE_PROJECT-cicd-Dev"
az ad app create --display-name "$APP_DISPLAY_NAME"
# Go to portal and look at Enterprise Application

# Set ENV Variables
DEV_AZURE_CLIENT_ID=$(az ad app list --query "[?displayName== '${APP_DISPLAY_NAME}'].appId" -o tsv)
echo "DEV_AZURE_CLIENT_ID: $DEV_AZURE_CLIENT_ID"

DEV_APPLICATION_ID=$(az ad app list --query "[?displayName== '${APP_DISPLAY_NAME}'].id" -o tsv)
echo "DEV_APPLICATION_ID: $DEV_APPLICATION_ID"
```

```bash
# Create service principal for App
az ad sp create --id $DEV_AZURE_CLIENT_ID
DEV_SERVICE_PRINCIPAL_ID=$(az ad sp show --id $DEV_AZURE_CLIENT_ID --query id -o tsv)
echo "DEV_SERVICE_PRINCIPAL_ID=$DEV_SERVICE_PRINCIPAL_ID"
```

```bash
GITHUB_ENV=Dev
REPO_WITHOUT_SCHEME=$(git config --get remote.origin.url | grep -oP 'https://github.com/\K\S+')
echo "REPO_WITHOUT_SCHEME: $REPO_WITHOUT_SCHEME"


# Create dev environment in Github

gh api --method PUT -H "Accept: application/vnd.github+json" repos/$REPO_WITHOUT_SCHEME/environments/Dev
gh secret set AZURE_CLIENT_ID --body "${DEV_AZURE_CLIENT_ID}" --env Dev 

# CHange this
FEDERATED_ID_NAME=< federnated name i.e. ADE-CICD>
# Could be replaced by az ad app federated-credential create
# Create the federated identity credential for Dev
az rest --method POST \
    --uri "https://graph.microsoft.com/beta/applications/$DEV_APPLICATION_ID/federatedIdentityCredentials" \
    --body "{\"name\":\"$FEDERATED_ID_NAME\",\"issuer\":\"https://token.actions.githubusercontent.com\",\"subject\":\"repo:$REPO_WITHOUT_SCHEME:environment:$GITHUB_ENV\",\"description\":\"$GITHUB_ENV\",\"audiences\":[\"api://AzureADTokenExchange\"]}"


# To list after creating
az ad app federated-credential show --id "$DEV_AZURE_CLIENT_ID" --federated-credential-id  $FEDERATED_ID_NAME
```

```bash
RESOURCE_GROUP=rg-dev-center
PROJECT_ID=$(az devcenter admin project show -n $AZURE_PROJECT --query id -o tsv -g $RESOURCE_GROUP)
echo "PROJECT_ID=$PROJECT_ID"

# Assign reader role to project
az role assignment create \
    --scope "${PROJECT_ID}" \
    --role Reader \
    --assignee-object-id $DEV_SERVICE_PRINCIPAL_ID \
    --assignee-principal-type ServicePrincipal

# Give Dev Env User 
# Sometimes doesn't work, may have to set scope to project
az role assignment create \
    --scope "${PROJECT_ID}/environmentTypes/Dev" \
    --role "Deployment Environments User" \
    --assignee-object-id $DEV_SERVICE_PRINCIPAL_ID \
    --assignee-principal-type ServicePrincipal
```



```bash
# Trigger Pipeline by creating a new branch.
BRANCH=new-repo
git checkout -b $BRANCH
git push --set-upstream origin $BRANCH

# Skipping CICD protection rules 
# Skipping Branch Protection Rules
# Skipping Required Reviewers
```


```bash
# Check Actions
ACTIONS_URL="$(git config --get remote.origin.url)/actions"
echo "Click here! $ACTIONS_URL"
```

```bash
## DOES NOT WORK.. GO TO PORTAL
## Clean up, or trigger env delete
REPO_ID=$(echo $(gh api -H "Accept: application/vnd.github+json" repos/$REPO_WITHOUT_SCHEME) | jq .id)
echo $REPO_ID

ENV_NAME=ci-branch-$BRANCH-$REPO_ID
echo $ENV_NAME
az devcenter dev environment delete -d "$AZURE_DEVCENTER" --project "$AZURE_PROJECT" -n "$ENV_NAME" -y
```
