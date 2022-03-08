# Stardog on Azure Container Instances.


## Limitations

* You can only mount Azure Files shares to Linux containers. Review more about the differences in feature support  for Linux and Windows container groups in the [overview](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-overview#linux-and-windows-containers).
* Azure file share volume mount requires the Linux container run as *root* .
* Azure File share volume mounts are limited to CIFS support.

> [!NOTE]
> Mounting an Azure Files share to a container instance is similar to a Docker [bind mount](https://docs.docker.com/storage/bind-mounts/). If you mount a share into a container directory in which files or directories exist, the mount obscures files or directories, making them inaccessible while the container runs.

> [!IMPORTANT]
> If you are deploying container groups into an Azure Virtual Network, you must add a [service endpoint](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-service-endpoints-overview) to your Azure Storage Account.



## Prerequisites
* A Stardog license file
* An Azure Subscription and a Resource Group already created.


## Step 1 - Create an Azure file share

Before using an Azure file share with Azure Container Instances, you must create it. Run the following script to create a storage account to host the file share, and the share itself. The storage account name must be globally unique, so the script adds a random value to the base string.

```azurecli-interactive

# Change these six parameters as needed   

ACI_PERS_RESOURCE_GROUP=stardogResourceGroup 
ACI_PERS_STORAGE_ACCOUNT_NAME=stardogsa$RANDOM 
ACI_PERS_LOCATION=eastus
ACI_PERS_STARDOG_HOME_SHARE_NAME=stardoghome
ACI_PERS_STARDOG_EXT_SHARE_NAME=stardogext
STARDOG_CONTAINER_NAME=stardog-demo

# Create the storage account with the parameters
    az storage account create \
        --resource-group $ACI_PERS_RESOURCE_GROUP \
        --name $ACI_PERS_STORAGE_ACCOUNT_NAME \
        --location $ACI_PERS_LOCATION \
        --sku Standard_LRS
        
# Create the file shares

#stardog home file share
az storage share create \
    --name $ACI_PERS_STARDOG_HOME_SHARE_NAME \
    --account-name $ACI_PERS_STORAGE_ACCOUNT_NAME

#stardog divers extentions divers
az storage share create \
    --name $ACI_PERS_STARDOG_EXT_SHARE_NAME \
    --account-name $ACI_PERS_STORAGE_ACCOUNT_NAME

```

## Step 2 - Get storage credentials

To mount an Azure file share as a volume in Azure Container Instances, you need three values: the storage account name, the share name, and the storage access key.

* **Storage account name** - If you used the preceding script, the storage account name was stored in the `$ACI_PERS_STORAGE_ACCOUNT_NAME` variable. To see the account name, type:

  ```console
  echo $ACI_PERS_STORAGE_ACCOUNT_NAME
  ```

* **Share name** - This value is already known (defined as `stardoghome` and `stardogext`  in the preceding script)

* **Storage account key** - This value can be found using the following command:

  ```azurecli-interactive
  STORAGE_KEY=$(az storage account keys list --resource-group $ACI_PERS_RESOURCE_GROUP --account-name $ACI_PERS_STORAGE_ACCOUNT_NAME --query "[0].value" --output tsv)
  echo $STORAGE_KEY
  ```

## Step 3 - Upload stardog license file, stardog properties file and drivers to Stardog Home File Share

### Upload stardog license change change STARDOG_LICENSE_PATH value if is necesary.

```azurecli-interactive
#Update STARDOG_LICENSE_PATH value with your stardog license file location.

STARDOG_LICENSE_PATH=stardog-license-key.bin

az storage file upload \
    --account-name $ACI_PERS_STORAGE_ACCOUNT_NAME \
    --account-key $STORAGE_KEY \
    --share-name $ACI_PERS_STARDOG_HOME_SHARE_NAME  \
    --source $STARDOG_LICENSE_PATH \
    --path "stardog-license-key.bin"

```

### Generate stardog.properties file

```shell-script
cat >stardog.properties <<EOL
# Flag to enable the cluster, without this flag set, the rest of the properties have no effect
pack.enabled=false

# enable BI/SQL
sql.server.enabled=true
sql.server.port=5806
sql.server.commit.invalidates.schema=true

EOL
```
### Upload stardog.properties file

```azurecli-interactive
az storage file upload \
    --account-name $ACI_PERS_STORAGE_ACCOUNT_NAME \
    --account-key $STORAGE_KEY \
    --share-name $ACI_PERS_STARDOG_HOME_SHARE_NAME  \
    --source "stardog.properties" \
    --path "stardog.properties"
```

### Optional - Upload Extentions Drivers.

Upload extentions drivers if need them.

```azurecli-interactive
# Example set the values and uncomment DRIVER_LOCAL_PATH and DRIVER_NAME.

# DRIVER_LOCAL_PATH=driver/SparkJDBC.jar
# DRIVER_NAME=SparkJDBC-SparkJDBC42-2.6.11.1014.jar

az storage file upload \
    --account-name $ACI_PERS_STORAGE_ACCOUNT_NAME \
    --account-key $STORAGE_KEY \
    --share-name $ACI_PERS_STARDOG_EXT_SHARE_NAME  \
    --source $DRIVER_LOCAL_PATH \
    --path $DRIVER_NAME
```


## Step 4 - Generate stardog deployable yaml file
```shell

cat >stardog-container-demo.yaml <<EOL
apiVersion: '2019-12-01'
location: eastus
name: ${STARDOG_CONTAINER_NAME}
properties:
  containers:
  - name: stardog
    properties:
      environmentVariables: 
      - name: STARDOG_HOME
        value: '/var/opt/stardog' 
      - name: STARDOG_PROPERTIES
        value: '/var/opt/stardog/stardog.properties'
      - name: STARDOG_EXT
        value: '/var/opt/stardog-ext'
      - name: STARDOG_SERVER_JAVA_ARGS
        value: '-Xmx2g -Xms2g -XX:MaxDirectMemorySize=4g'
      image: stardog/stardog:latest
      ports:
      - port: 5820
      - port: 5806
      resources:
        requests:
          cpu: 2
          memoryInGB: 8
      volumeMounts:
      - mountPath: /var/opt/stardog
        name: stardog-home
      - mountPath: /var/opt/stardog-ext 
        name: stardog-ext
  osType: Linux
  restartPolicy: Always
  ipAddress:
    type: Public
    ports:
      - port: 5820
      - port: 5806
    dnsNameLabel: ${STARDOG_CONTAINER_NAME}
  volumes:
  - name: stardog-home
    azureFile:
      sharename: ${ACI_PERS_STARDOG_HOME_SHARE_NAME}
      storageAccountName: ${ACI_PERS_STORAGE_ACCOUNT_NAME}
      storageAccountKey: ${STORAGE_KEY}
  - name: stardog-ext
    azureFile:
      sharename: ${ACI_PERS_STARDOG_EXT_SHARE_NAME}
      storageAccountName: ${ACI_PERS_STORAGE_ACCOUNT_NAME}
      storageAccountKey: ${STORAGE_KEY}
tags: {}
type: Microsoft.ContainerInstance/containerGroups

EOL

```

## Step 5 - Deploy stardog container demo on Azure container

``` azurecli-interactive
az container create --resource-group $ACI_PERS_RESOURCE_GROUP --file stardog-container-demo.yaml

```

### View deployment state

To view the state of the deployment, use the following az container show command:

``` azurecli-interactive
az container show --resource-group $ACI_PERS_RESOURCE_GROUP --name $STARDOG_CONTAINER_NAME --output table
```

### View container logs

View the log output of a container using the az container logs command. The --container-name argument specifies the container from which to pull logs. 

``` azurecli-interactive
az container logs --resource-group $ACI_PERS_RESOURCE_GROUP --name $STARDOG_CONTAINER_NAME
```
