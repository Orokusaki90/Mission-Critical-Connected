# Getting started with AlwaysOn Private

This guide walks you through the required steps to deploy AlwaysOn in a private version. The private version locks down all traffic to the deployed Azure services to come in through Private Endpoints only. Only the actual user traffic is still flowing in through the public ingress point of [Azure Front Door](https://azure.microsoft.com/services/frontdoor/#overview).

This deployment mode provides even tighter security but requires the use of self-hosted, VNet-integrated Build Agents. Also, for any debugging etc. users must connect through Azure Bastion and Jump Servers which can have an impact on developer productivity. **Be aware of these impacts before deciding to deploy AlwaysOn in private mode.**

![AlwaysOn Private Mode Architecture](/docs/media/Architecture-Foundational-Private.png)

## Overview

On a high level, the following steps will be executed:

1. Import Azure DevOps pipeline which deploys self-hosted Build Agents
1. Run the new pipeline to deploy the Virtual Machine Scale Sets for the Build Agents as well as Jump Servers and other supporting resources
1. Configure the self-hosted Build Agents in Azure DevOps
1. Set required variables in the variables files to reference the self-hosted Build Agent resources to later be able to create Private Endpoints

## Import pipeline to deploys self-hosted Build Agents

To deploy the infrastructure for the self-hosted Agents and all supporting services such as Jump Servers and private DNS zones, a ready-to-use Terraform template plus the corresponding ADO Pipeline is included in this repository.

> The following steps assume that you have already followed the general [Getting Started guide](/docs/reference-implementation/Getting-Started.md). If you have not done so yet, please go there first.

1. The ADO pipeline definition resides together with the other pipelines in `/.ado/pipelines`. It is called `azure-deploy-private-build-agents.yaml`. Start by importing this pipeline in Azure DevOps.

    ```bash
    # set the org/project context
    az devops configure --defaults organization=https://dev.azure.com/<your-org> project=<your-project>

    # import a YAML pipeline
    az pipelines create --name "Azure.AlwaysOn Deploy Build Agents" --description "Azure.AlwaysOn Build Agents" \
                        --branch main --repository https://github.com/<your-fork>/ --repository-type github \
                        --skip-first-run true --yaml-path "/.ado/pipelines/azure-deploy-private-build-agents.yaml"
    ```

    > You'll find more information, including screenshots on how to import and manage YAML-based pipelines in the overall [Getting Started Guide](./Miscellaneous-Getting-Started.md).

1. Now locate the Terraform variables files for the Build Agent deployment at `/src/infra/build-agents/variables.tf`. Adjust the values as required for your use case. For instance, you might want to change the deployment location. If you (later) want to change the SKU size of the VMs for Build Agents and/or Jump Servers, those settings are maintained in the respective `vmss-*.tf` template files in the same directory.

1. If you already know that you have special requirements regarding the software that needs to be present on the Build Agents to build your application code, go modify the `cloudinit.conf` in the same directory.

    > Please note that our self-hosted agents do not include the same [pre-installed software](https://docs.microsoft.com/azure/devops/pipelines/agents/hosted) as the Microsoft-hosted agents. Also, our Build Agents are only deployed as Linux VMs. You can technically change to Windows agents, but this is out of scope for this guide.

1. Commit your changes in Git and make sure to push them to your repository. It can be on the `main` branch but this is not required. You can later select which branch to deploy from.

## Set pipeline variables

Before we can deploy the private build agent and the private version of AlwaysOn, we need to update our pipeline variables so that the pipeline, and Terraform, knows to now include Private Endpoints for the self-hosted Build Agents and lock down all other traffic. This also applies to the Terraform state storage account.

The templates are fully prepared for this, we only need to set three additional variables.

1. Locate the variables file(s) for the respective environments in the `/.ado./pipelines/config` directory. E.g `/.ado/pipelines/config/variables-values-e2e.yaml`. (Adjust this, based on which environment your are configuring right now).
1. Fill out these variables:

```yaml
- name:  'buildAgentTerraformResourceGroup'
  value: 'terraformstate-rg'   # <=== Change this if needed!
- name:  'buildAgentTerraformStorageAccount'
  value: 'ao2e2etfstatestoreba' # <=== Change this! This needs to be a globally unique name and is only used to store the Terraform state of the private build agents themselves.
```

3. Make sure to update all the values shown above to reflect your environment! You have noted down the values earlier when you checked the provisioned resources.
3. Commit the changes to git and push them.

## Deploy self-hosted Build Agent infrastructure

Now that the pipeline for the self-hosted Agent infrastructure is imported and the settings adjusted, we are ready to deploy it. Note that this is done using the Microsoft-hosted agents. We have no requirement here yet for a self-hosted agent (plus, it would create a chicken-and-egg problem anyway).

This also creates a second Terraform state storage account which is only being used for this private build agent infrastructure. Since it needs to be deployed by Microsoft-hosted agents, this storage account cannot be created with Private Endpoints. The state storage accounts which are being used for the actual deployments, however, are then protected by Private Endpoints so that only the private build agents can access them.

1. Run the previously imported pipeline. Make sure to select the right branch. Select `e2e` as the environment. You can repeat the same steps later for `int` and `prod` when you are ready to use them.

    ![Run pipeline with environment selector](/docs/media/run-pipeline-with-environment-selector.png)

1. Wait until the pipeline is finished before you continue.

1. Go through the Azure Portal to your newly created Resource Group (something like `aoe2ebuildagents-rg`) to see all the resources that were provisioned for you.
1. Take a note of the name of the Resource Group. You will need it in the next step.
1. Also, note down the name of the VNet that was provisioned within this Resource Group, something like `aoe2ebuildagents-vnet`.

    ![self-hosted agent resources in azure](/docs/media/self-hosted-agents-resources-in-azure.png)

## Update pipelines variables

After we have deployed the private build agents, we need to update the pipeline so that Terraform will later know where to deploy Private Endpoints for the build agents.

1. Locate the variables file(s) for the respective environments in the `/.ado./pipelines/config` directory. E.g `/.ado/pipelines/config/variables-values-e2e.yaml`. (Adjust this, based on which environment your are configuring right now).
1. Find and fill out the following lines based on the output of the previous step:

```yaml
- name: 'buildAgentResourceGroupName'
  value: 'aoe2ebuildagents-rg' # <=== Change this!
- name: 'buildAgentVnetName'
  value: 'aoe2ebuildagents-vnet' # <=== Change this!
```

3. Commit the changes to git and push them.

## Configure self-hosted Build Agents in ADO

Next step is to configure our newly created Virtual Machine Scale Set (VMSS) as a self-hosted Build Agent pool in Azure DevOps. ADO will from there on control most operations on that VMSS, like scaling up and down the number of instances.

1. In Azure DevOps navigate to your project settings
1. Go to `Agent pools`
1. Add a pool and select as Pool type `Azure virtual machine scale set`
1. Select your `e2e` Service Connection and locate the VMSS.

    > **Important!** Make sure to select the scale set which ends on `-buildagents-vmss`, not the one for the Jump Servers!

1. Set the name of the pool to `e2e-private-agents` (adjust this when you create pools for other environments like `int`)
1. Check the option `Automatically tear down virtual machines after every use`. This ensures that every build run executes on a fresh VM without any leftovers from previous runs
1. Set the minimum and maximum number of agents based on your requirements. We recommend to start with a minimum of `0` and a maximum of `6`. This means that ADO will scale the VMSS down to 0 if no jobs are running to minimize costs.
1. Click Create

    ![Self-hosted Agent Pool in ADO](/docs/media/self-hosted-agents-pool-in-ado.png)

    > Setting the minimum to `0` saves money by starting build agents on demand, but can slow down the deployment process.

## Deploy AlwaysOn in private mode

Now everything is in place to deploy the private version of AlwaysOn. Just run your deployment pipeline, for example for the E2E environment. You might notice a longer delay until the job actually starts. This is due to the fact the ADO first needs to spin up instances in the scale set before they can pick up any task.

Otherwise you should see no immediate difference in the deployment itself. However, when you check the deployed resources, you will notice differences. For example that AKS is now deployed as a private cluster or that you will not be able to see the repositories in the Azure Container Registry through the Azure Portal anymore (due to the network restrictions to only allow Private Endpoint traffic).

## Use Jump Servers to access the deployment

In order to access the now locked-down services like AKS or Key Vault, you can use the Jump Servers which were provisioned as part of the self-hosted Build Agent deployment.

1. First we need to fetch the password to log on to the Jump Servers. The password is stored in the Key Vault inside the Build Agent resource group
1. Open the Key Vault in the Azure Portal and navigate to the Access Policies blade. Add an access policy for yourself (and maybe other users as well).

    ![Access Policy](/docs/media/private_build_agent_keyvault.png)
1. Then go to the Secrets blade, and retrieve the value of the secret `vmadmin-secret`
1. Next, navigate to the Jump Server VMSS in the same resource group. E.g. `aoe2ebuildagents-jumpservers-vmss`, open the Instances blade and select one of the instances (there is probably only one)
    ![Jump Server instances](/docs/media/private_build_agent_jumpservers_instances.png)
1. Select the Bastion blade, enter `adminuser` as username and the password that you copied from Key Vault. Click Connect.
1. You now have established an SSH connection via Bastion to the Jump Server which has a direct line of sight to your private resources.
    ![SSH jump server](/docs/media/private_build_agent_jumpserver_ssh.png)
1. Use for example `az login` and `kubectl` to connect to and debug your resources.

---

[Back to documentation root](/docs/README.md)