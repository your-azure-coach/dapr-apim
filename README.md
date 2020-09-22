This repo contains the technical instructions to complete an end-to-end scenario for using Dapr with Azure API Management Self-Hosted Gateway.  More information can be found on this [blog](https://yourazurecoach.com/blog).

## Setup your environment

* Install the [REST Client extension](https://marketplace.visualstudio.com/items?itemName=humao.rest-client) for Visual Studio Code

* Install the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)

* Install the [Dapr CLI](https://github.com/Dapr/cli#getting-started)

* Install version 0.10.0 of the Dapr runtime to get started.

```
Dapr init --kubernetes --runtime-version 0.10.0
```

* Clone this repo

```
git clone https://github.com/your-azure-coach/Dapr-logic-apps.git
```

## Create Azure resources

These resources in Azure are needed for testing Dapr integration with them

* Open the command prompt

* Log into Azure CLI and follow the instructions

```
az login
```

* Create an Azure resource group

```
az group create --name dapr-apim --location westeurope
```

* Create an Azure API Management instance

> If you have already an existing API Management instance (*Developer or Premium tier*), you can also use that one, as the provisioning of an API Management instance takes between 15 and 30 minutes

```
az apim create --name dapr-apim --resource-group dapr-apim --location westeurope --publisher-email toon@yourazurecoach.com --publisher-name YourAzureCoach
```

* Choose a unique Azure storage account name

```
set STORAGE_ACCOUNT_NAME=<YOUR_UNIQUE_STORAGE_ACCOUNT_NAME>
```

* Create an Azure storage account.  This will be used as a Dapr statestore

```
az storage account create --name %STORAGE_ACCOUNT_NAME% --resource-group dapr-apim
```

* Set storage account key variable

```
for /f %i in ('az storage account keys list --account-name %STORAGE_ACCOUNT_NAME% --query [0].value --output tsv') do set STORAGE_ACCOUNT_KEY=%i
```

* Choose a unique Azure Service Bus namespace name

```
set SERVICE_BUS_NAMESPACE_NAME=<YOUR_UNIQUE_SERVICE_BUS_NAMESPACE_NAME>
```

* Create an Azure Service Bus namespace.  This will be used for testing Dapr publish/subscribe.

```
az servicebus namespace create --name %SERVICE_BUS_NAMESPACE_NAME% --resource-group dapr-apim
```

* Create an Azure Service Bus topic, named *orders*.

```
az servicebus topic create --name orders --namespace-name %SERVICE_BUS_NAMESPACE_NAME% --resource-group dapr-apim
```

* Create a subscription for this topic, named order-processing

```
az servicebus topic subscription create --resource-group dapr-apim --namespace-name %SERVICE_BUS_NAMESPACE_NAME% --topic-name orders --name order-processing
```

* Set the service bus connection string

> :warning: for production usage, I highly recommend to create an application specific authorization rule

```
for /f %i in ('az servicebus namespace authorization-rule keys list --name RootManageSharedAccessKey --namespace-name %SERVICE_BUS_NAMESPACE_NAME% --resource-group dapr-apim --query primaryConnectionString --output tsv') do set SERVICE_BUS_CONNECTION_STRING=%i
```

## Register and install the self-hosted gateway

* Open your existing API Management instance - or create a new one

* Go to *Gateways* and add a new Gateway `local-apim-gateway`

* In the *Deployment* tab, choose `Kubernetes` and download the YAML file

* Open the YAML file, add the following Dapr annotations and *Save* it in the repository root

```yaml
kind: Deployment
spec:
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "local-apim-gateway-app"
```

> Thanks to these annotations, the Dapr control plane knows that it should inject a Dapr sidecar into the pod

* Update the deployment spec, to use the preview version of the self-hosted gateway and *Save*.

`mcr.microsoft.com/azure-api-management/gateway:1.1.0-preview-8`

* Execute the deployment commands as shown in the *Deployment* section in the Azure Portal

```cmd
kubectl create secret generic xxx --from-literal=value="xxx" --type=Opaque

kubectl apply -f local-apim-gateway.yaml
```

* Validate the deployment.
    * The pod is up and running
    * The Dapr side car is injected
    * The Gateway status is online 

```cmd
kubectl get pods
kubectl describe pod <pod-id>
kubectl logs <pod-id> <container-id>
```

## Create the Order API

* Create a new Order API in API Management:
  * **Display name**: ORDER API 
  * **API URL suffix**: sales 
  * **Gateways**: *select the `local-apim-gateway`* 
  * **Subscription required**: no

* Add a health check operation
  * **Name**: Get Health
  * **URL**: GET /heatlh

* Add this snippet to the inbound section of the operation policy

```xml
<inbound>
    <base />
    <return-response>
        <set-status code="200" reason="" />
        <set-body>@("Everything OK.  Greatings from " + context.Deployment.Region)</set-body>
    </return-response>
</inbound>
```

* Configure port-forwarding so we can invoke the API

```
kubectl port-forward service/local-apim-gateway 8080:80 8081:443
```

* Open the `requests/tests.http` file in Visual Studio Code and execute the **GET Health** request.

## Publish subscribe

* Create a secret for the Azure Service Bus connection string

> :warning: for production usage, I highly recommend to use a secret store like Azure Key Vault

```
kubectl create secret generic dapr-apim-service-bus --from-literal=connectionString=%SERVICE_BUS_CONNECTION_STRING%
```

* Create a Dapr publish subscribe component that points to Azure Service Bus

```
kubectl apply -f components/pub-sub-orders.yaml
```

* Add an API operation to the Order API in API Management: **POST Order**

`POST /order`

* Add this snippet to the inbound section of the operation policy

```xml
<inbound>
    <base />
    <publish-to-dapr topic="@("pub-sub-orders/orders")" timeout="10" ignore-error="false" template="liquid">{{body}}</publish-to-dapr>
    <return-response>
        <set-status code="200" />
    </return-response>
</inbound>
```

> You probably notice the strange expression for the topic name.  The reason is that the policy was built against Dapr v0.9.0 and we are using Dapr v0.10.0, which requires the pub sub component name to be part of URI.  Via the expression, I can bypass the naming limitations, which does not allow a '/'.

* Open the `requests/tests.http` file in Visual Studio Code and execute the **POST Order** request to create an order.

  *An order should arrive on the Service Bus "orders" subscription*
  
## Service invocation

* Run the HTTP BIN app as a Dapr application `httpbin-app`

```
kubectl apply -f services/httpbin-app.yaml
```

* Configure port forwarding, to we can easily test the HTTP BIN app

```
kubectl port-forward deployment/local-apim-gateway 3500:3500
``` 

* Open the `requests/prerequisites.http` file in Visual Studio Code and execute the **Test Service Invocation** request.  This just echos back the request. 

* Add an API operation to the Order API in API Management: **GET Orders**

`GET /order`

* Add this snippet to the inbound section of the operation policy

```xml
<inbound>
    <base />
    <set-backend-service backend-id="dapr" dapr-app-id="httpbin-app" dapr-method="get" />
    <set-header name="x-yac-test" exists-action="override">
        <value>request-header</value>
    </set-header>
</inbound>
```

* Open the `requests/tests.http` file in Visual Studio Code and execute the **Get Orders** request.  This just echos back the request. 

## State store

* Create a Dapr state store component that points to Azure Table Storage

```
kubectl apply -f components/state-store-orders.yaml
```

* Open the `requests/prerequisites.http` file in Visual Studio Code and execute the required commands with the REST Client extension.  This populates and validates the order state store.


* Add an API operation to the Order API in API Management: **GET Order By Id**

`GET /order/{id}`

* Add this snippet to the inbound section of the operation policy

```xml
<inbound>
    <base />
    <set-backend-service base-url="http://localhost:3500" />
    <rewrite-uri template="/v1.0/state/state-store-orders/{id}" copy-unmatched-params="true" />
</inbound>
```
> At the time of writing, there is no policy available for the state store.  However, we can achieve this, using the existing policies.

* Open the `requests/tests.http` file in Visual Studio Code and execute the **Get Order 1** and **Get Order 2** requests to get the order details.


## Cleanup

* Let's cleanup our Kubernetes environment

```
kubectl delete deployment local-apim-gateway
kubectl delete deployment httpbin-app
kubectl delete service local-apim-gateway
kubectl delete configmap local-apim-gateway-env
kubectl delete secret local-apim-gateway-token
kubectl delete component pub-sub-orders
kubectl delete component state-store-orders
```

* Let's remove the Azure resources

```
az group delete --name dapr-apim --yes
```
