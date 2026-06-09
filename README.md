# Project 1 - Static Website Hosting with Azure Blob Storage and Azure Front Door
 
## Overview
 
A static website deployed to Azure Blob Storage, distributed globally via Azure Front Door, secured with a managed SSL/TLS certificate, and routed through a custom domain. I have chosen to make a lightweight food travel blog to keep track of places I've tried. I used AI to create the HTML file but none of the photos or written content.
 
**Live site:** [www.thecurrytrail.com](https://www.thecurrytrail.com)
 
## What Was Built
 
- Azure Blob Storage account with static website hosting enabled
- Azure Front Door Standard profile with a custom endpoint
- Origin group and origin pointing at the Blob Storage static website endpoint
- HTTPS-enforced route with HTTP → HTTPS redirect
- Managed SSL/TLS certificate provisioned automatically by Front Door
- Custom domain validated via DNS TXT record and linked to the Front Door route
- CNAME record in registrar DNS pointing `www` at the Front Door endpoint
## Prerequisites
 
- Azure account with an active subscription
- Azure CLI installed and authenticated (`az login`)
- A registered domain name
## Deployment
 
### Step 1 - Create a Resource Group
 
```bash
az group create \
  --name <YOUR_RESOURCE_GROUP> \
  --location uksouth
```
 
### Step 2 - Create a Storage Account
 
```bash
az storage account create \
  --name <YOUR_STORAGE_ACCOUNT> \
  --resource-group <YOUR_RESOURCE_GROUP> \
  --location uksouth \
  --sku Standard_LRS \
  --kind StorageV2 \
  --allow-blob-public-access true
```
 
### Step 3 - Enable Static Website Hosting
 
In the Azure Portal:
 
1. Go to your storage account → **Data management** → **Static website**
2. Toggle to **Enabled**
3. Set **Index document name**: `index.html`
4. Set **Error document path**: `404.html`
5. Save — note the **Primary endpoint** URL
Or via CLI:
 
```bash
az storage blob service-properties update \
  --account-name <YOUR_STORAGE_ACCOUNT> \
  --static-website \
  --index-document index.html \
  --404-document 404.html
```
 
### Step 4 - Upload the Website
 
In the Portal: navigate to **Containers** → `$web` → **Upload**, set blob name to `index.html` and content type to `text/html`.
 
Or via CLI:
 
```bash
az storage blob upload \
  --account-name <YOUR_STORAGE_ACCOUNT> \
  --container-name '$web' \
  --file index.html \
  --name index.html \
  --content-type 'text/html'
```
 
### Step 5 - Create an Azure Front Door Profile
 
```bash
az afd profile create \
  --profile-name <YOUR_FD_PROFILE> \
  --resource-group <YOUR_RESOURCE_GROUP> \
  --sku Standard_AzureFrontDoor
```
 
### Step 6 - Create a Front Door Endpoint
 
```bash
az afd endpoint create \
  --endpoint-name <YOUR_ENDPOINT_NAME> \
  --profile-name <YOUR_FD_PROFILE> \
  --resource-group <YOUR_RESOURCE_GROUP>
```
 
Note the **Endpoint hostname** from the output — it will look like `<name>.z03.azurefd.net`.
 
### Step 7 - Create an Origin Group
 
```bash
az afd origin-group create \
  --origin-group-name <YOUR_ORIGIN_GROUP> \
  --profile-name <YOUR_FD_PROFILE> \
  --resource-group <YOUR_RESOURCE_GROUP> \
  --probe-request-type GET \
  --probe-protocol Https \
  --probe-interval-in-seconds 30 \
  --probe-path / \
  --sample-size 4 \
  --successful-samples-required 3
```
 
### Step 8 - Create an Origin
 
```bash
az afd origin create \
  --origin-name <YOUR_ORIGIN_NAME> \
  --origin-group-name <YOUR_ORIGIN_GROUP> \
  --profile-name <YOUR_FD_PROFILE> \
  --resource-group <YOUR_RESOURCE_GROUP> \
  --host-name <YOUR_STORAGE_ACCOUNT>.z33.web.core.windows.net \
  --origin-host-header <YOUR_STORAGE_ACCOUNT>.z33.web.core.windows.net \
  --https-port 443 \
  --http-port 80 \
  --priority 1 \
  --weight 1000
```
 
### Step 9 - Create a Route
 
```bash
az afd route create \
  --route-name <YOUR_ROUTE_NAME> \
  --profile-name <YOUR_FD_PROFILE> \
  --resource-group <YOUR_RESOURCE_GROUP> \
  --endpoint-name <YOUR_ENDPOINT_NAME> \
  --origin-group <YOUR_ORIGIN_GROUP> \
  --supported-protocols Https Http \
  --https-redirect Enabled \
  --forwarding-protocol HttpsOnly \
  --patterns-to-match "/*" \
  --link-to-default-domain Enabled
```
 
### Step 10 - Add a Custom Domain
 
```bash
az afd custom-domain create \
  --custom-domain-name <YOUR_CUSTOM_DOMAIN_NAME> \
  --profile-name <YOUR_FD_PROFILE> \
  --resource-group <YOUR_RESOURCE_GROUP> \
  --host-name www.<YOUR_DOMAIN> \
  --minimum-tls-version TLS12 \
  --certificate-type ManagedCertificate
```
 
Grab the DNS validation token:
 
```bash
az afd custom-domain show \
  --custom-domain-name <YOUR_CUSTOM_DOMAIN_NAME> \
  --profile-name <YOUR_FD_PROFILE> \
  --resource-group <YOUR_RESOURCE_GROUP> \
  --query "validationProperties"
```
 
Add the following records in your DNS registrar:
 
| Type  | Hostname     | Value                              |
|-------|--------------|------------------------------------|
| CNAME | www          | `<YOUR_ENDPOINT>.z03.azurefd.net`  |
| TXT   | _dnsauth.www | `<VALIDATION_TOKEN_FROM_ABOVE>`    |
 
### Step 11 - Link Custom Domain to Route
 
```bash
az afd route update \
  --route-name <YOUR_ROUTE_NAME> \
  --profile-name <YOUR_FD_PROFILE> \
  --resource-group <YOUR_RESOURCE_GROUP> \
  --endpoint-name <YOUR_ENDPOINT_NAME> \
  --custom-domains <YOUR_CUSTOM_DOMAIN_NAME>
```
 
Check validation status:
 
```bash
az afd custom-domain show \
  --custom-domain-name <YOUR_CUSTOM_DOMAIN_NAME> \
  --profile-name <YOUR_FD_PROFILE> \
  --resource-group <YOUR_RESOURCE_GROUP> \
  --query "domainValidationState"
```
 
Full propagation usually takes up to 30 minutes.
 
## Cleanup
 
You can delete the resource group in the Portal, or run this CLI command:
 
```bash
az group delete \
  --name <YOUR_RESOURCE_GROUP> \
  --yes --no-wait
```
 
This will delete all resources, assuming they are all in the same resource group.
 
## Issues Encountered
 
- **Subscription propagation delay** — a newly created Pay As You Go subscription was not immediately accessible via the Azure CLI, returning `SubscriptionNotFound` errors. Resolved by being patient and coming back to work on this after a coffee.
- **CDN region error** — Azure Front Door profiles must be deployed to the `global` region; I had used `uksouth` without realising.
- **Origin group health probe** — the `--probe-path /` parameter is required when health probes are enabled; omitting it causes a validation error.
## Learning Objectives
 
- Static website hosting on Azure Blob Storage
- Global content delivery with Azure Front Door Standard
- Managed SSL/TLS certificate provisioning via Front Door
- Custom domain validation using DNS TXT records

 ## Cost Management
 - Setting up a budget alert is sensible to manage the cost of the project. This is mostly from azure front door rather than the storage account.
 - A more cost effective alternative to using front door would be to setup a free cloudflare account.
