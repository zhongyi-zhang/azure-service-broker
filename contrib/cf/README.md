# Installing the Azure Service Broker on Cloud Foundry

The Azure Service Broker is an [Open Service Broker](https://wwww.openservicebrokerapi.org)-compatible application for provisioning and managing services in Microsoft Azure. This document describes how to deploy it on [Cloud Foundry](https://cloudfoundry.org).

## Prerequisites

What you will need:

- **Cloud Foundry environment**: there are multiple ways to use [Cloud Foundry on Azure](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/cloudfoundry-get-started).
- **Azure CLI**: You can [install it locally](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) or use it in the [Azure Cloud Shell](https://docs.microsoft.com/en-us/azure/cloud-shell/overview?view=azure-cli-latest)

- **Cloud Foundry CLI**: You can [install it locally](https://docs.cloudfoundry.org/cf-cli/install-go-cli.html) or use it in the [Azure Cloud Shell](https://docs.microsoft.com/en-us/azure/cloud-shell/overview?view=azure-cli-latest).

## Create an Azure Redis Cache

The Azure Service Broker uses Redis as a backing store for its state. Create a cache using the Azure CLI:

```console
az redis create -n asb-cache -g myresourcegroup -l <location> --sku Basic --vm-size C1 --enable-non-ssl-port
```

Note the `hostName` and `primaryKey` in the output as these will be needed later.

## Obtain Your Subscription ID

```console
$ az account show --query id
```

## Create a Service Principal

The Azure Service Broker uses a service principal to provision Azure resources on your behalf.

```console
$ az ad sp create-for-rbac
```

The new service principal will be assigned, by default, to the `Contributor`
role. The output of the command above will be similar to the following:

```console
{
  "appId": "redacted",
  "displayName": "azure-cli-xxxxxx",
  "name": "http://azure-cli-xxxxxx",
  "password": "redacted",
  "tenant": "redacted"
}
```

## Update the Cloud Foundry manifest

Open contrib/cf/manifest.yml and enter the values obtained in the earlier steps:

```yaml
---
  name: asb
  buildpack: go_buildpack
  command: broker 
  env:
    AZURE_SUBSCRIPTION_ID: <YOUR SUBSCRIPTION ID>
    AZURE_TENANT_ID: <TENANT ID FROM SERVICE PRINCIPAL>
    AZURE_CLIENT_ID: <APPID FROM SERVICE PRINCIPAL>
    AZURE_CLIENT_SECRET: <PASSWORD FROM SERVICE PRINCIPAL>
    LOG_LEVEL: DEBUG
    REDIS_HOST: <HOSTNAME FROM AZURE REDIS CACHE>
    REDIS_PASSWORD: <PRIMARYKEY FROM AZURE REDIS CACHE>
    AES256_KEY: AES256Key-32Characters1234567890
    BASIC_AUTH_USERNAME: username
    BASIC_AUTH_PASSWORD: password
    GOPACKAGENAME: github.com/Azure/azure-service-broker
    GO_INSTALL_PACKAGE_SPEC: github.com/Azure/azure-service-broker/cmd/broker
```

**IMPORTANT**: The default values for `AES256_KEY`, `BASIC\_AUTH\_USERNAME`, and `BASIC\_AUTH\_PASSWORD` should never be used in production environments.

## Push the broker to Cloud Foundry

Once you have added the necessary environment variables to the CF manifest, you can simply push the broker:

```console
cf push -f contrib/cf/manifest.yml
```

## Register the Service Broker with Cloud Foundry

With the broker app deployed, the final step is to register it as a service broker in Cloud Foundry. Note that this step must be executed by a Cloud Foundry administrator unless you are using the `--space-scoped` flag to limit it to a single CF space.

```console
cf create-service-broker azure-service-broker username password https://asb.apps.example.com
```