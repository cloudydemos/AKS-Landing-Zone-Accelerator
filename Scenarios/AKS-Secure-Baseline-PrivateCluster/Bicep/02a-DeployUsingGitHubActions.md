# AKS Cluster Deployment via GitHub Actions using OpenID Connect and Bicep (IaC)

For this approach, we will be using GitHub Actions using OpenID Connect and Infrastructure-as-Code (IaC) using Bicep to deploy the AKS cluster, to derive following benefits:

* Infrastructure-as-Code (IaC) - Infrastructure is defined as code, and can be version controlled and reviewed. 
* OpenID Connect - OpenID Connect is an authentication protocol that allows you to connect securely to Azure resources using your GitHub account.
* GitHub Actions - GitHub Actions is a feature of GitHub that allows you to automate your software development workflows.
* Bicep - Bicep is a Domain Specific Language (DSL) for deploying Azure resources declaratively. It aims to drastically simplify the authoring experience with a cleaner syntax, improved type safety, and better support for modularity and code re-use.

This will require performing the following tasks:

1. Forking this repository into your GitHub account
2. Configuring OpenID Connect in Azure
3. Setting Github Actions secrets
4. Triggering the first GitHub Actions workflow

### Forking this repository into your GitHub account

* Fork this [repository](https://github.com/Azure/AKS-DevSecOps-Workshop) into your GitHub account by clicking on the "Fork" button at the top right of its page. Use the default name "AKS-Landing-Zone-Accelerator" for this fork in your repo.


### Verify Subscription

Login to Azure using the first command below, and then when you run the second command verify that this is the correct subscription where your deployment will be created. Use az account set --name "subscription name" if necessary in order to select the right one.

```bash
az login
az account show
```

### Configuring OpenID Connect in Azure

1. Create an Azure AD application Used to deploy the IaC to your Azure Subscription

   ```bash
   uniqueAppName=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c10 ; echo '')
   echo $uniqueAppName
   appId=$(az ad app create --display-name $uniqueAppName --query appId --output tsv)
   echo $appId
   ```

2. Create a service principal for the Azure AD app.

   ```bash
   assigneeObjectId=$(az ad sp create --id $appId --query id --output tsv)
   echo $assigneeObjectId 
   ```

3. Create a role assignment for the Azure AD app. This gives that app contributer access to the currently selected subscription.

   ```bash
   subscriptionId=$(az account show --query id --output tsv)
   echo $subscriptionId
   az role assignment create --role contributer --subscription $subscriptionId --assignee-object-id  $assigneeObjectId --assignee-principal-type ServicePrincipal --scope /subscriptions/$subscriptionId
   ```

4. Create a Personal Access Token (PAT) for your repo in GitHub. This is used to create a self-hosted GitHub runner, which is in turn used to deploy code to your cluster. Follow these instructions to create a PAT: [https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token] The scope needs to include "read:org", "repo" and "workflow". The value is used in the next section.

5. Open the file "cloud-init.yml" and find the token "username" on line 40. Replace "username" with the name of your GitHub account (the text after github.com in your GitHub repo). This is the repo where your self-hosted runner will be created.

6. Get the ID of the "AKS Devs" Group you created in the [Earlier Step](./02-aad.md)

   ```bash
   AKSUSERACCESSPRINCIPALID=$(az ad group show --group "AKS Admins" --query id --output tsv)
   echo $AKSUSERACCESSPRINCIPALID
   ```

7. Register Resource Providers

   ```bash
   az feature register --namespace "Microsoft.Network" --name "AzureFirewallBasic"
   az feature register --namespace "Microsoft.ContainerService" --name "AKS-AzureKeyVaultSecretsProvider"
   az feature register --name "EnablePodIdentityPreview" --namespace "Microsoft.ContainerService"
   az feature register --namespace "Microsoft.ContainerService" --name "AKS-AzureKeyVaultSecretsProvider"
   az feature register --namespace "Microsoft.Compute" --name "EncryptionAtHost"
   az feature register --namespace "Microsoft.ContainerService" --name "AKS-ExtensionManager"
   az feature register --namespace "Microsoft.ContainerService" --name "AKS-Dapr"
   az feature register --name "AKS-KedaPreview" --namespace "Microsoft.ContainerService"
            
   az provider register --namespace Microsoft.Network
   az provider register --namespace Microsoft.ContainerService
   az provider register --namespace Microsoft.OperationsManagement
   az provider register --namespace Microsoft.OperationalInsights
   ```

### Setting Github Actions secrets

1. Open your forked Github repository and click on the `Settings` tab.
2. In the left-hand menu, expand `Secrets and variables`, and click on `Actions`.
3. Click on the `New repository secret` button for each of the following secrets:
   * `AZURE_SUBSCRIPTION_ID`(this is the `subscriptionId`from the previous step)
   * `AZURE_TENANT_ID` (run `az account show --query tenantId --output tsv` to get the value)
   * `AZURE_CLIENT_ID` (this is the `appId` from the JSON output of the `az ad app create` command)
   * `CLUSTER_RESOURCE_GROUP` (this is the `resourceGroupName` from earlier step)
   * `VM_PW` (this is the password you want your VM to be set to)
   * `RUNNER_CFG_PAT` (this is the `Personal Access Token` from earlier step)
   * `AKSUSERACCESSPRINCIPALID` (this is the `AKSUSERACCESSPRINCIPALID` from earlier step)
   * `EMAIL` (your email address - used by LetsEncrypt to send notifications.)

### Triggering the GitHub Actions workflow

* Enable GitHub Actions for your repository by clicking on the "Actions" tab, and clicking on the `I understand my workflows, go ahead and enable them` button.
* To trigger the AKS deployment workflow manually:
  * click on the `Actions` tab.
  * Select `.github/workflows/1-deploy-hub.yml`.
  * Click on the `Run workflow` button and accept the default options.
  * This will trigger the first workflow. There are seven altogether and each will trigger the next until all seven are complete. This may take some time.

## Next Step

:arrow_forward: [Cleanup](./08-cleanup.md)
