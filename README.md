# About

Repo az-container-demo can be used to demo creating, maintaining, and deploying .NET Core apps natively and containerized on Azure PaaS services. The options for running containerized apps and the scenario that each is best suited for follows:

* Azure App Service: Best suited for monolithic app workloads that require HTTP-based access and the desire to not manage any infrastructure.
* Azure Container Apps: Best suited for stateless microservices workloads without having to manage any infrastructure or container orchestration.
* Azure Container Instance: Best suited for easy configuration and deployment of app container where uptime is not a priority.
* Azure Kubernetes Service: Best suited for running containerized workloads at scale, where all of the other options have failed to meet the need.

The .NET project is the standard To-Do app commonly used in Azure learning modules and repos. It is configured for ASP.NET Core and .NET Core 7.0. This repo can demonstrate the full application lifecycle, including development, test, deploy, and CI/CD.

NOTE: This demo assumes use of Powershell for the command line.  

## Prepare the app for deployment to Azure cloud servivces

### Clone the repo

Use `git clone` to clone the git repo locally on your and select the main branch

```console
git clone https://github.com/loublick-ms/az-container-demo
git branch -m main
```console

The project structure will look as follows:

```console
* AZ-CONTAINER-DEMO
  * todoapp
    * Controllers
    * Data
    * Migrations
    * Models
    * Properties
    * Views
    * wwwroot
  * .gitignore
  * Dockerfile
  * CONTRIBUTING.md
  * LICENSE
  * LICENSE.md
  * README.md
```

### Create a local database

Use the SQLite database for immediate testing of the code and any database schema changes.

Prepare the database by running the dotnet database migrations.

```console
dotnet ef database update
```

Run the app locally to test the code and database schema.

```console
dotnet run
```

### Create an Azure Resource Group

Create a resource group in Azure to contain all of the services required to complete this demo.

```console
az group create --name rg-container-demo --location "East US"
```

### Create an Azure SQL database server

Create an Azure SQL database server to be used by the app when deployed in the cloud.

```console
az sql server create --name dbs-container-demo --resource-group rg-container-demo --location "East US" --admin-user <db admin username> --admin-password <admin password>
```

### Create the Azure SQL database

Create the SQL Server database on the SQL Server that was just create and display the connection string. Copy and save the connection string for later use in configuring the app and the cloud services.

```console
az sql db create --resource-group rg-container-demo --server dbs-container-demo --name todoDB --service-objective S0
az sql db show-connection-string --client ado.net --server dbs-container-demo --name todoDB
```

### Update the C# code to connect to the Azure SQL database

Update the database context in Startup.cs to connect to the Azure SQL database instead of the local SQLite database

`services.AddDbContext<MyDatabaseContext>(options => options.UseSqlServer(Configuration.GetConnectionString("MyDbConnection")));`

### Update the .NET Entity Framework code to access the Azure SQL database

Delete the database migrations associated with the SQLite database.

```console
rm -r Migrations
```

Recreate migrations for Azure SQL

```console
dotnet ef migrations add InitialCreate
```

Create an Azure SQL database connection string environment variable in Powershell. This env variable will be used by the .NET Entity Framework `dotnet ef` command to initialize the Azure SQL datbase schema for the app.

```console
$env:ConnectionStrings:MyDbConnection=<database connection string>
```

Run the .NET database migrations for Azure SQL database to create the database schema.

```console
dotnet ef database update
```

Run the web app locally with the Azure SQL database.

```console
dotnet run
```

## Build a container image for the app and store it in Azure Container Registry

### Create a container image for the app

Create a Dockerfile that will be used to build the container image. The file should be located in the project root directory and include both the .NET restore and publish tasks, as show in the example below:

```console
# todoapp Dockerfile example
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /source

# Copy csproj and restore the project as distinct layers
COPY todoapp/*.csproj .
RUN dotnet restore --use-current-runtime  

# Copy the remaining app files and build app
COPY todoapp/. .
RUN dotnet publish -c Release -o /app --self-contained

# Final stage: build the container image
FROM mcr.microsoft.com/dotnet/aspnet:7.0
WORKDIR /app
COPY --from=build /app .
ENTRYPOINT ["dotnet", "todoapp.dll"]
```

### Create the container image using the Dockerfile

The image will be stored in your local Docker Desktop image registry. Run the `docker build` command from the project root directory.

```console
docker build -t todoapp .
```

Run the containerized app locally in Docker Desktop to test the app. Use the `docker run` command to run the container.

```console
docker run -it --rm -p 8000:80 --name todoapp todoapp
```

### Store the Docker image in Azure Container Registry

Create a container registry in Azure using the Azure CLI with the `az acr create` command. When naming your registry, you can only use alpha numeric characters.

```console
az acr create --name acrcontainerdemo --resource-group rg-container-demo --sku standard --admin-enabled true
```

Display the credentials of the registry you just created using `az acr credential show`. These credentials will be used to connect Docker to your container registry in Azure.

```console
az acr credential show --name acrcontainerdemo --resource-group rg-container-demo
```

Connect Docker to your registry using the `docker login` command. Use the credentials displayed by the `az acr credential show` command.

```console
docker login acrcontainerdemo.azurecr.io
```

Next, create the image alias in Docker that will be pushed to ACR. Before you push an image, you must create an alias for the image that specifies the repository and tag that the Docker registry creates. The repository name must be of the form
`<login_server>/<image_name>:<tag>`. Use the `docker tag` command to perform this operation.

```console
docker tag todoapp acrcontainerdemo.azurecr.io/todoapp:v1
```

If you run `docker image ls`, you will see two entries for the image: one with the original name, and the second with the new alias.

Next push the Docker container image to the registry using the `docker push` command.

```console
docker push acrcontainerdemo.azurecr.io/todoapp:v1
```

If you run the `az acr repository list` command, you will see the image in a list of all images. Alternatively, you can use the `az acr repository show` command to display the list of images for a specific repository.

```console
az acr repository list --name acrcontainerdemo.azurecr.io
az acr repository show --repository todoapp --name acrcontainerdemo.azurecr.io
```

## Deploy the image to an Azure Kubernetes Cluster

Run the app container image in an Azure Kubernetes Cluster.

### Create and configure AKS cluster

Create an AKS cluster using the `az aks create` command  with the `--enable-addons monitoring` and `--enable-msi-auth-for-monitoring` parameter to enable Azure Monitor Container insights with managed identity authentication.

```console
az aks create -g rg-container-demo -n kc-container-demo --attach-acr acrcontainerdemo --enable-managed-identity --node-count 1 --enable-addons monitoring --enable-msi-auth-for-monitoring  --generate-ssh-keys
```

Manage your new cluster with the command-line client kubectl. Download the AKS cluster credentials for use by kubectl.

```console
az aks get-credentials --resource-group rg-container-demo --name kc-container-demo
```

The credentials will be merged into the ~/.kube/config file.

Verify the credentials by running the `kubectl get` command. This command will return a list of nodes in the cluster.

```console
kubectl get nodes
```

### Deploy the app container image to the AKS cluster

Create a Kubernetes manifest file to define the desired state of the cluster such as which container images to run. The manifest file is written in yaml, as shown in the example below:

```console
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todoapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: todoapp
  template:
    metadata:
      labels:
        app: todoapp
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
      - name: todoapp
        image: acrcontainerdemo.azurecr.io/todoapp:v2
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: todoapp
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: todoapp
```

Apply the manifest file to the cluster to deploy the container image from Azure Container Registry. Test the app by accessing the public IP of the default AKS load balancer. It may take a couple of minutes for the app deployment to complete.

```console
kubectl apply -f .\todoapp.yaml
```

Test the app container running in AKS.

## Deploy the image to an Azure Container Instance

Run the app container image in an Azure Container Instance.

### Create an Azure Container Instance and deploy the container image

Deploy and run the app container in Azure Container Instance using the `az container create` command. Note that the first time 'az container create` is run, the command will create an Azure Container Instance service using the name provided in the command. Thereafter, you will use the same command to deploy updated container images using the tag to differentiate the images.

```console
az container create --resource-group rg-container-demo --name aci-container-demo --image acrcontainerdemo.azurecr.io/todoapp:latest --dns-name-label dns-container-demo --registry-username <registry username> --registry-password <registry password>
```

### Test the app container running in ACI using the Fully-Qualified Domain Name

The FQDN is located in the Azure portal in the ACI Overview blade.

## Deploy the image to an Azure App Service

Run the app container image in an Azure App Service.

### Create a user-managed identity and authorize it for access to the container registry

Create a managed identity to authenticate with the container registry using the `az identity` command.

```console
az identity create --name id-container-demo --resource-group rg-container-demo
```

Retrieve the managed identity principal ID and registry ID and authorize the principal to pull images from the container registry using the `az role assignment create` command.

```console
$principalId=$(az identity show --resource-group rg-container-demo --name id-container-demo --query principalId --output tsv)
$registryId=$(az acr show --resource-group rg-container-demo --name acrcontainerdemo --query id --output tsv)
az role assignment create --assignee $principalId --scope $registryId --role "AcrPull"
```

### Create the App Service that will run the app container

Create the App Service Plan that will host the App Service using the `az appservice plan create` command. The App Service Plan corresponds to the virtual machine hosts that will run the web app.

```console
az appservice plan create --name asp-container-demo --resource-group rg-container-demo --is-linux
```

Create the App Service using the `az webapp create` command.

```console
az webapp create --resource-group rg-container-demo --plan asp-container-demo --name wa-container-demo --deployment-container-image-name acrcontainerdemo.azurecr.io/todoapp:v2
```

Enable the user-assigned managed identity in the web app with the `az webapp identity assign` command.

```console
$id=$(az identity show --resource-group rg-container-demo --name id-container-demo --query id --output tsv)
az webapp identity assign --resource-group rg-container-demo --name wa-container-demo --identities $id
```

Configure your app to pull from Azure Container Registry by using managed identities with the `az resource update` command with the webapp configuration.

```console
$appConfig=$(az webapp config show --resource-group rg-container-demo --name wa-container-demo --query id --output tsv)
az resource update --ids $appConfig --set properties.acrUseManagedIdentityCreds=True
```

Set the client ID your web app uses to pull from Azure Container Registry. This step isn't needed if you use the system-assigned managed identity.

```console
$clientId=$(az identity show --resource-group rg-container-demo --name id-container-demo --query clientId --output tsv)
az resource update --ids $appConfig --set properties.AcrUserManagedIdentityID=$clientId
```

Enable CI/CD in the webapp by creating a webhook using the `az acr webhook create` command.

```console
$ciCdUrl=$(az webapp deployment container config --enable-cd true --name wa-container-demo --resource-group rg-container-demo --query CI_CD_URL --output tsv)
az acr webhook create --name cd-as-container-demo --registry acrcontainerdemo --uri $ciCdUrl --actions push --scope todoapp:latest
```

## DEVSECOPS --------------------------------------------------------------------------

Initialize environment variables.

```console
$RG_NAME='rg-containers-devsecops-demo'
$LOCATION='eastus'
$SUBSCRIPTION_ID=$(az account show --query id -o tsv)
$TENANT_ID="$(az account show --query tenantId --output tsv)"
echo $RG_NAME
echo $LOCATION
echo $SUBSCRIPTION_ID
echo $TENANT_ID
```

### Create an Azure Resource Group to contain the Azure services and the app

```console
az login
az group create --name $RG_NAME --location $LOCATION
```

### Configuring OpenID Connect in Azure

Create an Azure AD application.

```console
$uniqueAppName='app-name-containers-devsecops-demo'
echo $uniqueAppName
$appId=$(az ad app create --display-name $uniqueAppName --query appId --output tsv)
echo $appId
```

Create a service principal for the Azure AD app.

```console
$assigneeObjectId=$(az ad sp create --id $appId --query id --output tsv)
echo $assigneeObjectId 
```

Create a role assignment for the Azure AD app.

```console
az role assignment create --role owner --subscription $SUBSCRIPTION_ID --assignee-object-id  $assigneeObjectId --assignee-principal-type ServicePrincipal --scope /subscriptions/$SUBSCRIPTION_ID
```

Configure a federated identity credential on the Azure AD app.

You use workload identity federation to configure an Azure AD app registration to trust tokens from an external identity provider (IdP), such as GitHub.

In /todoapp-svc/credential.json file, replace `<your-github-username>` with your GitHub username (in your locally cloned repo).

"subject": "repo:`<your-github-username>`/az-containers-devsecops-demo:ref:refs/heads/main",

If you name your new repository something other than `az-containers-devsecops-demo`, you will need to replace az-containers-devsecops-demo above with the name of your repository. Also, if your deployment branch is not main, you will need to replace main with the name of your deployment branch.

Then run the following command from the root folder of the cloned repo to create a federated credential for the Azure AD app.

```console
az ad app federated-credential create --id $appId --parameters app/demo/todoapp-svc/credential.json
```

## Setting Github Actions secrets

Open your Github repository and click on the Settings tab. In the left-hand menu, expand Secrets and variables, and click on Actions. Click on the New repository secret button for each of the following secrets:

AZURE_SUBSCRIPTION_ID (this is the subscriptionIdfrom the previous step)
AZURE_TENANT_ID (run az account show --query tenantId --output tsv to get the value)
AZURE_CLIENT_ID (this is the appId from the JSON output of the az ad app create command)
CLUSTER_RESOURCE_GROUP (this is the RG_NAME from earlier step)

### Triggering the GitHub Actions workflow

Enable GitHub Actions for your repository by clicking on the “Actions” tab, and clicking on the I understand my workflows, go ahead and enable them button. To trigger the AKS deployment workflow manually:

* Click on the Actions tab.
* Select .github/workflows/services-deployment-workflow.yml.
* Click on the Run workflow button.

Alternatively, you can make a change to the azure-services.bicep file (e.g. change the clusterName parameter), and push the change to your Github repo. This will trigger the GitHub Actions workflow.

### Connect to your cluster

To connect to your cluster, use the following commands:

```console
$CLUSTER_NAME=aks-containers-devsecops
$RG_NAME=rg-containers-devsecops-demo
az aks get-credentials --name $CLUSTER_NAME --resource-group $RG_NAME --admin
kubectl get nodes
````

## Enabling Workload Identity

Note As of Feb. 2023, this feature is in public preview, with expectations that GA is soon, so the following ‘Register preview providers’ section will not be required once the feature is GA.

Set the following environment variables in your bash session by updating the values and executing in your terminal.

```console
$RG_NAME='rg-containers-devsecops-demo'
$LOCATION='eastus'
$CLUSTER_NAME='aks-containers-devsecops'
$SUBSCRIPTION_ID=$(az account show --query id -o tsv)
$KEYVAULT_NAME=$(az keyvault list -g $RG_NAME --query "[0].name" -o tsv)
$KEYVAULT_SECRET_NAME='kvsncontainersdevsecops'
```

### Register preview providers on your subscription

Run the following command to register the preview provider feature for Workload Identity.

```console
az feature register --namespace "Microsoft.ContainerService" --name "EnableWorkloadIdentityPreview"
```

Wait for the feature to be enabled by running this command, the state should show “Registered” when complete. This may take up to 10 minutes.

```console
az feature show --namespace "Microsoft.ContainerService" --name "EnableWorkloadIdentityPreview"
```

Once the Workload Identity preview feature is registered, you must register the parent Microsoft.ContainerService resource provider.

```console
az provider register --namespace Microsoft.ContainerService
```

Add and/or update the aks-preview extension with the Azure CLI.

```console
az extension add --name aks-preview 
az extension update --name aks-preview
```

## Enable ODIC and Workload Identity on the AKS Cluster

Execute the following CLI command to enable oidc-issuer and to enable workload identity on your AKS cluster. This operation will take several minutes.

```console
az aks update --resource-group $RG_NAME --name $CLUSTER_NAME --enable-oidc-issuer --enable-workload-identity
```

The OIDC Issuer feature allows Azure Active Directory (Azure AD) or other cloud provider identity and access management platforms to discover the API server’s public signing keys. The Azure AD Workload Identity feature for Kubernetes integrates with the capabilities native to Kubernetes to federate with external identity providers. It allows for workloads in your AKS cluster to make use of AAD Managed Identities.

Set the ODIC Issuer URL to a variable for usage later.

```console
$AKS_OIDC_ISSUER='$(az aks show -n $CLUSTER_NAME -g $RG_NAME --query "oidcIssuerProfile.issuerUrl" -otsv)'
echo $AKS_OIDC_ISSUER
echo ($AKS_OIDC_ISSUER + ".well-known/openid-configuration")
```

Verify that you now see a mutating webhook pod on your cluster. The mutating admission webhook is used to project a signed service account token to a workload’s volume and inject environment variables to pods.

```console
az aks get-credentials --resource-group $RG_NAME  --name $CLUSTER_NAME --admin --overwrite
kubectl get pods -n kube-system
```

### Create a managed identity and grant permission to Azure Keyvault

Create Managed Identity in your resource group.

```console
$USER_ASSIGNED_IDENTITY_NAME="containers-devsecops-id"
az identity create --name "$USER_ASSIGNED_IDENTITY_NAME" --resource-group "$RG_NAME" --location "$LOCATION" --subscription "$SUBSCRIPTION_ID"

Create Access Policy against Keyvault, allowing the identity to get secrets.

```console
$USER_ASSIGNED_CLIENT_ID="$(az identity show --resource-group "$RG_NAME" --name "$USER_ASSIGNED_IDENTITY_NAME" --query 'clientId' -o tsv)"
az keyvault set-policy --name "$KEYVAULT_NAME" --secret-permissions get --spn "$USER_ASSIGNED_CLIENT_ID" --resource-group "$RG_NAME"
```

Create a Secret in Key Vault.

```console
$USERID=$(az ad signed-in-user show --query id -o tsv)
az keyvault set-policy --name "$KEYVAULT_NAME" --secret-permissions list set get --object-id $USERID --resource-group "$RG_NAME"
az keyvault secret set --vault-name "$KEYVAULT_NAME" --name "$KEYVAULT_SECRET_NAME" --value "Containers DevSecOps demo\!"

### Create a Service Account and Establish Federated Identity

Create environmennt variables used to create the service account and Federated Identity by executing the following.

```console
$SERVICE_ACCOUNT_NAME="workload-identity-sa"
$SERVICE_ACCOUNT_NAMESPACE="default"
```

Create a YAML file to deploy a K8S service account in the app/demo/todoapp-svc folder. Replace the user assigned client ID created previously for the Key Vault access policy, and the service account name and namespace. Note the annotations and labels required for this service account to leverage Workload Identity.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: "<USER_ASSIGNED_CLIENT_ID>"
  labels:
    azure.workload.identity/use: "true"
  name: "<SERVICE_ACCOUNT_NAME>"
  namespace: "<default>"
EOF
```

Deploy K8S service account to Azure.

```console
kubectl apply -f todoapp-svc/
```

Establish Federated Identity. The namespace and service account name are used to create the subject identifier in the federation. Once this is setup, this Managed Identity will now trust tokens coming from our Kubernetes cluster. The subject claim identifies the principal that will be the subject of the token.

```console
az identity federated-credential create --name myfederatedIdentity --identity-name "$USER_ASSIGNED_IDENTITY_NAME" --resource-group "$RG_NAME" --issuer "$AKS_OIDC_ISSUER" --subject system:serviceaccount:"$SERVICE_ACCOUNT_NAMESPACE":"$SERVICE_ACCOUNT_NAME"
```

After the federation is setup, navigate to your cluster resource group, and you will now see an identity. Click the Identity resource and select the “Federated credentials” blade under Settings.

### Deploy a sample workload and test

Deploy a sample .NET application to test the Workload Identity. You can find source code for different programming languages that implement MSAL and KeyVault integration [here](<https://github.com/Azure/azure-workload-identity/tree/main/examples>).

The following YAML deploys a sample .NET application that writes to the log the content of the secret inside keyvault. The .NET application expects two environment variables for the Kevault URL and the Keyvault secret name references. You can find source code for different programming languages that implement MSAL and KeyVault integration here.

Note the following required annotations in the K8S YAML configuration:

* azure.workload.identity/use: “true”
* serviceAccountName: <SERVICE_ACCOUNT_NAME>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: quick-start
  namespace: "default""
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: "workload-identity-sa"
  containers:
    - image: ghcr.io/azure/azure-workload-identity/msal-net
      name: oidc
      env:
      - name: KEYVAULT_URL
        value: https://akv-containers-devsecops.vault.azure.net/
      - name: SECRET_NAME
        value: kvsncontainersdevsecops
```

Execute the following to retrieve the Key Vault URL.

```console
export KEYVAULT_URL="$(az keyvault show -g $RG_NAME -n $KEYVAULT_NAME --query properties.vaultUri -o tsv)"
```

Once the pod is running, ensure the pod is showing the KeyVault secret:

```console
kubectl logs quick-start
```

If the pod communication to the Key Vault was successful, you will see the following message: Pod Logs

>START 03/16/2023 18:59:11 (quick-start)
>Your secret is Containers DevSecOps demo\!

Inspect the additional environment variables and volumeMounts created:

```console
kubectl describe pod quick-start
```

You should notice the new environment variables and volume mount.

Additionally, you can inspect the token mounted to the pod by executing into the quick-start container. The path here is found in the volumeMount. You can further copy this token and paste it into jwt.io to inspect the content of the token.

```console
kubectl exec quick-start -- cat /var/run/secrets/azure/tokens/azure-identity-token
```
