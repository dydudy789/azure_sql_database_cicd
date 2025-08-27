# Azure SQL Database Project (SSDT) + CI/CD with Azure DevOps #

This repo contains a SQL Database Project (.sqlproj) built with SSDT and a pipeline that builds a DACPAC and deploys it to Dev and Prod Azure SQL Databases using Azure DevOps.

## Repository layout ##
```
terraform_project_sqldb/
├─ src/                           # .sql object files (tables, views, sprocs, etc.)
├─ terraform_project_sqldb.sqlproj
├─ db-pipeline.yml                # Azure DevOps pipeline (build + deploy)
└─ README.md
```

## Prerequisites ##

* Azure DevOps organization and project
* Azure subscription with two Azure SQL Servers + Databases (Dev & Prod)
* VS Code with SQL Database Projects extension
* An Azure Resource Manager service connection in Azure DevOps

## Azure DevOps setup

### 1) Service connection (ARM)
- **Project Settings → Service connections → New → Azure Resource Manager → _Service principal (automatic)_**
- Select your **subscription** and **scope** (Subscription or a specific RG).
- Name it (e.g., `sc-azure-sub`).

### 2) Environments
- Create environments: **dev** and **prod**.
- *(Optional)* Add **approvals/checks** on `prod`.

### 3) Set Entra ID (AAD) admin on SQL Servers
Use an AAD **user or group** as the SQL Server admin.
```
  # DEV
az sql server ad-admin create \
  --resource-group "<rg-dev>" \
  --server "<sql-server-dev>" \
  --display-name "<AAD user or group display name>" \
  --object-id "<AAD objectId>"

# PROD
az sql server ad-admin create \
  --resource-group "<rg-prod>" \
  --server "<sql-server-prod>" \
  --display-name "<AAD user or group display name>" \
  --object-id "<AAD objectId>"
```



### 4) Grant the pipeline Service Principal rights in each database

**Goal:** create an AAD user for the pipeline’s Service Principal and grant `db_owner`.

**Steps**

1. Connect to the **target DB** (not `master`) as the SQL Server **AAD admin**.
2. Create the AAD user for the SP:

```
-- Run in the target DB (not master). Replace with your SP's display name.
CREATE USER [<service-principal-display-name>] FROM EXTERNAL PROVIDER;
ALTER ROLE db_owner ADD MEMBER [<service-principal-display-name>];
```
* Find the SP’s display name/objectId in Entra ID (App registrations → Enterprise applications).

### 5) Pipeline variables (Library → Variable Groups)

Create a group (e.g., sql-db-vars) with:

```
AZDO_SP_APPID = <service-principal-appId>
TENANT_ID     = <tenantId>

DEV_SQL_SERVER  = <sql-server-dev>.database.windows.net
DEV_SQL_DB      = <dev-db-name>
PROD_SQL_SERVER = <sql-server-prod>.database.windows.net
PROD_SQL_DB     = <prod-db-name>
```
* The service connection securely stores the SP secret—do not add secrets to variables.


## How the CI/CD pipeline works (detailed comments in yml deployment code) ##

### Build stage ###

* Restore packages if needed (legacy packages.config)

* Build the .sqlproj → produce a DACPAC

* Publish the DACPAC as a pipeline artifact

### Deploy to Dev ###

* Download the DACPAC artifact

* Acquire an AAD access token for https://database.windows.net via the service connection

* Run sqlpackage with /AccessToken: to publish to Dev



### Deploy to Prod ###

* (Optional) Manual approval on the prod environment

* Repeat the same publish step against Prod



<img width="1874" height="992" alt="db-deployment-pipelines-toggle" src="https://github.com/user-attachments/assets/e33e492c-5925-4994-94a4-269a049c51e4" />

<img width="1893" height="891" alt="db-deployment-pipelines-success" src="https://github.com/user-attachments/assets/2fd7bed4-3944-4f76-8875-6074a0a97b69" />





