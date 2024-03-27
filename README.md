### Install Bicep

Run the following command to ensure you have the latest version of Bicep:

```bash
az bicep install && az bicep upgrade
```

### Set Defaults

Run the following command to set the defaults. Throughout the setup, we will use these defaults and passing them as parameters to different components:

```bash
export LOCATION=uksouth
export RESOURCE_GROUP_NAME=rate-limiting-sample-rg
export TOPIC_NAME=rate-limiting-sample-fifo-topic
export TOPIC_SUBSCRIPTION_NAME=rate-limiting-sample-fifo-topic-subs
az configure --defaults location=$LOCATION group=$RESOURCE_GROUP_NAME
```

### Deploy the template to Azure

Run the following command to deploy the Bicep template to Azure. The command can take a minute or two to complete, and then you'll see a successful deployment.

```bash
make provision
```

To see the deployment details:

```bash
az deployment group list --output table --resource-group $RESOURCE_GROUP_NAME
```

### Clean up deployment

Run the following command to remove the resource group and all resources associated with it.

```bash
az group delete --resource-group $RESOURCE_GROUP_NAME
```

### Running Local

**Before you run in your local, complete steps in [Infrastructure Provisioning](#infrastructure-provisioning) section.**

You need [Minikube](https://minikube.sigs.k8s.io/docs/start/) and [Skaffold](https://skaffold.dev/) installed in your environment for local development. You also need [Helm](https://helm.sh/docs/intro/install/) installed to install [Redis](https://redis.io/) in your local cluster. Redis is used as a distributed lock provider for the message handlers.

**Step 1: Creating Local Cluster:**

Create a new [minikube](https://minikube.sigs.k8s.io/docs/start/) cluster:

```bash
mingw32-make setup-local-cluster
```

**Step 2: Setting up Azure Service Bus Connection:**

Assuming you have provisioned the resources in previous steps on [Infrastructure Provisioning](#infrastructure-provisioning) section, you need to create a new `.env` file and save it in the root directory of the project. The `.env` file should contain the following environment variables:

```bash
SERVICE_BUS_CONNECTION_STRING=<your-service-bus-connection-string>
SERVICE_BUS_TOPIC_NAME=<your-service-bus-topic-name>
SERVICE_BUS_SUBSCRIPTION_NAME=<your-service-bus-subscription-name>
```

To generate the connection string, you can use the following command:

```bash
mingw32-make setup-local-env
```

**Step 3: Deploying Secrets:**

Run the following command to create a generic secret in the cluster:

```bash
mingw32-make setup-local-cluster-secret
```

**Step 4: Deploying Services:**

Run the following command to deploy the services to your cluster using [Skaffold](https://skaffold.dev/):

```bash
mingw32-make deploy-local-cluster
```

By convention, `message handler service` will use `rate-limiting-api:8081` as the local address for the rate limiting API inside the cluster. This address is set in the environment variables for the service.

**Step 5: Running Load Test:**

Run the following command to run the load test. This will generate ~15.5K messages and send them to the service bus via APIM. `message handler service` will listen to these messages and post to `rate limiting api`.

During this test, we expect the service bus to receive all the messages. However, `rate limiting api` will only receive **~80 messages per minute**. This rate limit is set in the `rate limiting api` and can be changed by updating the `application.yaml` file in the `rate limiting api` project.

```bash
mingw32-make load-test-local
```
