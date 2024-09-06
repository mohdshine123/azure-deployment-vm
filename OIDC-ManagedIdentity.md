# azure-deployment-vm
Configure OpenID Connect (OIDC) in Azure@# subscriptions
Configure OpenID Connect (OIDC) in Azure@# subscriptions

Overview
OpenID Connect (OIDC) allows your GitHub Actions workflows to access resources in Azure, without needing to store the Azure credentials as long-lived GitHub secrets.

This guide gives an overview of how to configure Azure to trust GitHub's OIDC as a federated identity, and includes a workflow example for the azure/login action that uses tokens to authenticate to Azure and access resources.

Configuration
To configure the OIDC in Azure, you will need to perform the following configuration.

Create a service principal or a user-assigned Managed Identity.
Add federated credentials.
Create GitHub secrets for storing Azure configuration.
1. Create Service Principal with contributor permissions (upon request)
Request Azure@# platform in ServiceNow using Azure Technical Assistance offering to:

Create a Service Principal for your subscription.

Associate Service Principal to the subscription as Contributor.
Add federated credentials with the detail described in the section below.
(OR) 

Create a user-assigned Managed Identity (self-service)
Follow the Azure documentation to create a user-assigned managed identity. Then, add the managed identity to the #Contributors group of your subscription.

2. Add Federated Credentials
Under the Managed Identity, or Service Principal, create a Federated Credential to trust your GitHub repository. The OIDC trust relationship created between GitHub and Azure is specific to the repo and cloud subscription. Provide the following details:

Organization: GitHub organization 
Repository name: without the organization;
Entity (ref) type: either Environment, Branch, Pull Request or Tag;
GitHub name: name of the environment, branch, pull request or tag, where wildcars (*) are allowed;
Name: free-form id of the federated credential.
To know more about how to create federated credential for Service Principal please follow this Azure documentation and for Managed Identity Please go through this Azure documentation.

3. Create GitHub secrets for storing Azure configuration
Store the following GitHub Secrets at repository lever or in the appropriate environment level:

AZURE_CLIENT_ID of the Service Principal or Managed Identity,
AZURE_TENANT_ID,
AZURE_SUBSCRIPTION_ID.
Update GitHub Actions workflow for OIDC
To update your workflows for OIDC, you will need to make two changes to your YAML:

Add permissions settings for the token.

The job or workflow run requires a permissions setting with id-token: write.

Use the azure/login action to exchange the OIDC token (JWT) for a cloud access token.

Example 1 : Authenticate to Azure
Create a GitHub Actions workflow in the organization and repository allowed, and use azure/login official action from GitHub's Marketplace.

jobs:
  azure-auth:
    runs-on: ubuntu-latest
    permissions:
      id-token: write # This is required for requesting the JWT
      contents: read  # This is required for actions/checkout
    steps:
      - name: Azure authentication
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - name: 'Run az commands'
        run: |
          az account show
          az group list
The above example exchanges an OIDC ID token with Azure to receive an access token, which can then be used to access cloud resources.

Example 2 : Retrieve the secret from Azure key-vault using reusable workflow
To use reusable workflows with OIDC to standardize and security harden your deployment steps, you need to edit your federated credential. Below is one example, you have to provide your details.

federatedidentityedit
To know more about how OIDC works with reusable workflow, follow this documentation .

To have Access control established to the azure key vault, federated credential should have a role assigned as “key vault secret user” to read or access the sources in the key vault. To do that you can refer the below example.

roleassign
Then create a secret in azure key-vault. You can refer the below screenshot for how to do that.

secret
Once all the above process have been completed, then create a reusable workflow. Below is one snippet.

vault.yml

name: Run Azure Login with OIDC and retrieve secret from azure key-vault.

on:
  workflow_call:
    inputs:
      AZURE_KEYVAULT_NAME:
        description: 'The name of the Azure Key Vault'
        type: string
        required: true
      AZURE_SECRET_NAME:
        description: 'The name of the secret in the Azure Key Vault'
        type: string
        required: true
    secrets:
      AZURE_SUBSCRIPTION_ID:
        description: 'The Azure subscription ID'
        required: true
      AZURE_CLIENT_ID:
        description: 'The Azure client ID'
        required: true
      AZURE_TENANT_ID:
        description: 'The Azure tenant ID'
        required: true
    outputs:
      vaultsecret:
        description: 'The secret from the Azure Key Vault'
        value: ${{ jobs.azure-login-and-get-secret.outputs.vaultsecret }}

permissions:
  id-token: write
  contents: read

jobs:
  azure-login-and-get-secret:
    runs-on: ubuntu-latest

    outputs:
      vaultsecret: ${{ steps.getsecret.outputs.SECRET }}

    steps:
      - name: Checkout
        uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11

      - name: Azure Login
        uses: azure/login@cb79c773a3cfa27f31f25eb3f677781210c9ce3d
        with:
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}

      - name: Get Secret
        id: getsecret
        uses: azure/cli@4db43908b9df2e7ac93c8275a8f9a448c59338dd
        with:
          inlineScript: |
            SECRET=$(az keyvault secret show --vault-name ${{ inputs.AZURE_KEYVAULT_NAME }} --name ${{ inputs.AZURE_SECRET_NAME }} --query value)

            echo "SECRET=$SECRET" >> $GITHUB_OUTPUT

            # echo "::add-mask::$SECRET"
            echo "The secret is: ${SECRET}"
The above reusable workflow will login into Azure through OIDC, then retrieve a secret from Azure key vault by using key-vault name and secret name.

Finally create your workflow which will use the above reusable one and print the secret which will be retrived from key-vault .Here is the example.

demo.yml

name: Use reusable workflow to use the secret.

on:
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  oidc:
    uses: /.github/workflows/vault.yml@main
    with:
      AZURE_KEYVAULT_NAME: ${{ vars.AZURE_KEYVAULT_NAME }}
      AZURE_SECRET_NAME: ${{ vars.AZURE_SECRET_NAME }}
    secrets:
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}

  demo:
    runs-on: ubuntu-latest
    needs: oidc

    steps:
      - name: Print the secret
        run: |
          echo "The secret is: ${{ needs.oidc.outputs.vaultsecret }}"
