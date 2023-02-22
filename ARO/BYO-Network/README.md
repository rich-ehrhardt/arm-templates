# A basic template for a Azure Red Hat OpenShift cluster for deploying onto an existing virtual network

This guide uses the Azure CLI tools. It is also possible to use the same template with the portal directly or PowerShell.

## Prerequisites

- Have an Azure subscription with user administrator access
- The subscription should have at least the following services resource providers registered:
    - Microsoft.Networks
    - Microsoft.Compute
    - Microsoft.Storage
    - Microsoft.RedHatOpenShift
    - Microsoft.Authorization
- Have the Azure CLI tools downloaded locally
- (Optional) Have `jq` installed if you want to use this to obtain some of the parameters per the below steps
- Have a Red Hat account if you are going to access the Red Hat marketplace post installation
- Have an existing virtual network within a resource group with the following minimum specifications
    - a control / master subnet with at least 4 IP addresses available
    - a worker subnet with at least the worker count plus one IP addresses available
    - routing that allows outbound communication to the internet to reach the Red Hat binaries (refer OpenShift documentation for specific URLs)

## Execution steps

1. Clone the repository and change to the ARO base directory if not already there.
    ```shell
    git clone https://github.com/rich-ehrhardt/arm-templates
    cd ARO/BYO-Network
    ```

2. (Optional) Obtain a Red Hat OpenShift pull secret
    This allows the installation of operators from the Red Hat Marketplace. Refer [here](https://console.redhat.com/openshift/install/pull-secret) to obtain a pull secret. Save this pull secret to a file on your local machine.

    Note that the pull secret can also be added after the cluster has been created by following [these instructions](https://learn.microsoft.com/en-us/azure/openshift/howto-add-update-pull-secret)

3. Export the key parameters 
    This step allows the parameters to be defined once as you follow steps.
    ```shell
    export NAME_PREFIX="<name_prefix>"
    export RG_NAME="<resource_group_name>"
    export LOCATION="<resource_group_location>"
    export PULL_SECRET=$(cat <pull_secret_file_name>)
    export VNET_NAME="<vnet_name>"
    export MASTER_SUBNET="<master_subnet_name>"
    export WORKER_SUBNET="<worker_subnet_name>"
    ```
    where 
    - `<name_prefix>` is the identifier which will prefix created resources. Needs to start with a lower case letter and be between 3 and 10 characters in length
    - `<resource_group_name>` is the name of the existing resource group. This should always start with the a lower case letter
    - `<resource_group_location>` is the Azure location to for the resource group
    - `<pull_secret_file_name>` is the full path and filename to a file containing the Red Hat OpenShift pull secret you saved in step 2
    - `<vnet_name>` is the name of the virtual network within the resource group to be used to deploy Red Hat OpenShift
    - `<master_subnet_name>` is the name of a subnet within the virtual network to be used for the control / master nodes
    - `<worker_subnet_name>` is the name of a subnet within the virtual network to be used for the worker nodes


        To obtain a list of available locations for your subscription, run the following.
        ```shell
        az account list-locations -o table
        ```

4. Create a service principal for the cluster
    This is used by the cluster to provision resources (such as scaling the worker nodes).  Note that you can only have one cluster registered to a service principal, so it is recommended to create a new service principal for each cluster.
    1. Use the following commands to create a service principal.
        ```shell
        az ad sp create-for-rbac --name <name> | tee ./sp-details.json
        ```
        where `<name>` is the name to give the new service provider.

        **NOTE** Take note of the contents into a secure location and then delete the `sp-details.json` file once the below steps are completed as it contains the service provider security details.

    2. The following parameters for the build can be obtained from the output of this creation.
        ```shell
        export CLIENT_ID=$(cat ./sp-details.json | jq -r '.appId')
        export CLIENT_SECRET=$(cat ./sp-details.json | jq -r '.password')
        export CLIENT_OBJECT_ID=$(az ad sp show --id $CLIENT_ID --query "id" -o tsv)
        ```

5. Obtain the Azure Red Hat Openshift resource provider object id
    This is required to give the resource provider access to change the virtual network.
    To obtain the object id, run the following:
    ```shell
    export RP_OBJECT_ID=$(az ad sp list --display-name "Azure Red Hat OpenShift RP" --query "[0].id" -o tsv)
    ```

6. Create the deployment
    This will create the deployment in the resource group you just created.
    ```shell
    az deployment group create \
    --name aro_deployment \
    --resource-group $RG_NAME \
    --template-file ./mainTemplate.json \
    --parameters namePrefix=$NAME_PREFIX \
    --parameters spClientId=$CLIENT_ID \
    --parameters spClientSecret=$CLIENT_SECRET \
    --parameters spObjectId=$CLIENT_OBJECT_ID \
    --parameters rpObjectId=$RP_OBJECT_ID \
    --parameters pullSecret=$PULL_SECRET \
    --parameters vnetName=$VNET_NAME \
    --parameters controlSubnetName=$MASTER_SUBNET \
    --parameters workerSubnetName=$WORKER_SUBNET
    ```

## Clean up Steps

When you are finished with the environment. Perform the following steps to remove it. 

**Note** This will delete all data and settings. Only do this when the environment is no longer required.

1. Delete the ARO Cluster
    1. Find the cluster details
        ```shell
        az aro list -o table
        ```
    2. Delete the cluster
        ```shell
        az aro delete -n <cluster_name> -g <resource_group>
        ```
        where
        - `<cluster_name>` is the name of the cluster shown in the table
        - `<resource_group>` is the resource group name shown in the table

2. Delete the service principal
    ```shell
    az ad sp delete --id <appId>
    ```
    where
    - `<appId>` is the service principal appId or client id that was created in the build process. If the file still exists, it is available in the `sp-details.json` file from the build process.

3. If not already, delete the `sp-details.json` file.
    ```shell
    rm ./sp-details.json
    ```