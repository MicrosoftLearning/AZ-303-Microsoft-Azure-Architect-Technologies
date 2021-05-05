---
lab:
    title: '12: Protecting Hyper-V VMs by using Azure Site Recovery'
    module: 'Module 12: Manage Workloads in Azure'
---

# Lab: Protecting Hyper-V VMs by using Azure Site Recovery
# Student lab manual

## Lab scenario

While Adatum Corporation has, over the years, implemented a number of high availability provisions for their on-premises workloads, its disaster recovery capabilities are still insufficient to address the Recovery Point Objectives (RPOs) and Recovery Time Objectives (RTOs) demanded by its business. Maintaining the existing secondary on-premises site requires an extensive effort and incurs significant costs. The failover and failback procedures are, for the most part, manual and are poorly documented. 

To address these shortcomings, the Adatum Enterprise Architecture team decided to explore capabilities of Azure Site Recovery, with Azure taking on the role of the hoster of the secondary site. Azure Site Recovery automatically and continuously replicates workloads running on physical and virtual machines from the primary to the secondary site. Site Recovery uses storage-based replication mechanism, without intercepting application data. With Azure as the secondary site, data is stored in Azure Storage, with built-in resilience and low cost. The target Azure VMs are hydrated following a failover by using the replicated data. The Recovery Time Objectives (RTO) and Recovery Point objectives are minimized since Site Recovery provides continuous replication for VMware VMs and replication frequency as low as 30 seconds for Hyper-V VMs. In addition, Azure Site Recovery also handles orchestration of failover and failback processes, which, to large extent, can be automated. It is also possible to use Azure Site Recovery for migrations to Azure, although the recommended approach relies on Azure Migrate instead.

The Adatum Enterprise Architecture team wants to evaluate the use of Azure Site Recovery for protecting on-premises Hyper-V virtual machines to Azure VM.

## Objectives
  
After completing this lab, you will be able to:

-  Configure Azure Site Recovery

-  Perform test failover

-  Perform planned failover

-  Perform unplanned failover


## Lab Environment

Windows Server admin credentials

-  User Name: **Student**

-  Password: **Pa55w.rd1234**

Estimated Time: 120 minutes


## Lab Files

-  \\\\AZ303\\AllFiles\\Labs\\12\\azuredeploy30312suba.json


### Exercise 0: Prepare the lab environment

The main tasks for this exercise are as follows:

1. Deploy an Azure VM by using an Azure Resource Manager QuickStart template

1. Configure nested virtualization in the Azure VM


#### Task 1: Deploy an Azure VM by using an Azure Resource Manager QuickStart template

1. From your lab computer, start a web browser, navigate to the [Azure portal](https://portal.azure.com), and sign in by providing credentials of a user account with the Owner role in the subscription you will be using in this lab.

1. In the Azure portal, open **Cloud Shell** pane by selecting on the toolbar icon directly to the right of the search textbox.

1. If prompted to select either **Bash** or **PowerShell**, select **PowerShell**. 

    >**Note**: If this is the first time you are starting **Cloud Shell** and you are presented with the **You have no storage mounted** message, select the subscription you are using in this lab, and select **Create storage**. 

1. In the toolbar of the Cloud Shell pane, select the **Upload/Download files** icon, in the drop-down menu select **Upload**, and upload the file **\\\\AZ303\\AllFiles\Labs\\12\\azuredeploy30312suba.json** into the Cloud Shell home directory.

1. From the Cloud Shell pane, run the following to create a resource groups (replace the `<Azure region>` placeholder with the name of the Azure region that is available for deployment of Azure VMs in your subscription and which is closest to the location of your lab computer):

   ```powershell
   $location = '<Azure region>'
   New-AzSubscriptionDeployment `
     -Location $location `
     -Name az30312subaDeployment `
     -TemplateFile $HOME/azuredeploy30312suba.json `
     -rgLocation $location `
     -rgName 'az30312a-labRG'
   ```

      > **Note**: To identify Azure regions where you can provision Azure VMs, refer to [**https://azure.microsoft.com/en-us/regions/offers/**](https://azure.microsoft.com/en-us/regions/offers/)

1. In the Azure portal, close the **Cloud Shell** pane.

1. From your lab computer, open another browser tab, navigate to the [301-nested-vms-in-virtual-network Azure QuickStart template](https://github.com/Azure/azure-quickstart-templates/tree/master/demos/nested-vms-in-virtual-network) and select **Deploy to Azure**. This will automatically redirect the browser to the **Hyper-V Host Virtual Machine with nested VMs** blade in the Azure portal.

1. On the **Hyper-V Host Virtual Machine with nested VMs** blade in the Azure portal, specify the following settings (leave others with their default values):

    | Setting | Value | 
    | --- | --- |
    | Subscription | the name of the Azure subscription you are using in this lab |
    | Resource group | **az30312a-labRG** |
    | Host Public IP Address Name | **az30312a-hv-vm-pip** |
    | Virtual Network Name | **az30312a-hv-vnet** |
    | Host Network Interface1Name | **az30312a-hv-vm-nic1** |
    | Host Network Interface2Name | **az30312a-hv-vm-nic2** |
    | Host Virtual Machine Name | **az30312a-hv-vm** |
    | Host Admin Username | **Student** |
    | Host Admin Password | **Pa55w.rd1234** |

1. On the **Hyper-V Host Virtual Machine with nested VMs** blade, select **Review + create** and then select **Create**.

    > **Note**: Wait for the deployment to complete. The deployment might take about 10 minutes.

#### Task 2: Configure nested virtualization in the Azure VM

1. In the Azure portal, search for and select **Virtual machines** and, on the **Virtual machines** blade, select **az30312a-hv-vm**.

1. On the **az30312a-hv-vm** blade, select **Networking**. 

1. On the **az30312a-hv-vm | Networking** blade, select **az30312a-hv-vm-nic1** and then select **Add inbound port rule**.

    >**Note**: Make sure that you modify the settings of **az30312a-hv-vm-nic1**, which has the public IP address assigned to it.

1. On the **Add inbound security rule** blade, specify the following settings (leave others with their default values) and select **Add**:

    | Setting | Value | 
    | --- | --- |
    | Destination port range | **3389** |
    | Protocol | **Any** |
    | Name | **AllowRDPInBound** |

1. On the **az30312a-hv-vm** blade, select **Overview**. 

1. On the **az30312a-hv-vm** blade, select **Connect**, in the drop-down menu, select **RDP**, and then click **Download RDP File**, after the RDP file downloads select **Open file** and select **Connect**.

1. When prompted, sign in with the following credentials:

-  User Name: **Student**

-  Password: **Pa55w.rd1234**

1. Within the Remote Desktop session to **az30312a-hv-vm**, in the Server Manager window, click **Local Server**, click the **On** link next to the **IE Enhanced Security Configuration** label, and, in the **IE Enhanced Security Configuration** dialog box, select both **Off** options.

1. Within the Remote Desktop session to **az30312a-hv-vm**, start Internet Explorer, navigate to the download page of [Microsoft Edge](https://www.microsoft.com/en-us/edge/business/download), download Microsoft Edge installer and perform the installation. 

1. Within the Remote Desktop session to **az30312a-hv-vm**, use Microsoft Edge to browse to [Windows Server Evaluations](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2019), and download the Windows Server 2019 **VHD** file to the **F:\VHDs** folder (you will need to create it first). 

1. Within the Remote Desktop session to **az30312a-hv-vm**, start **Hyper-V Manager**. 

1. In the **Hyper-V Manager** console, select the **az30312a-hv-vm** node, select **New** and, in the cascading menu, select **Virtual Machine**. This will start the **New Virtual Machine Wizard**. 

1. On the **Before You Begin** page of the **New Virtual Machine Wizard**, select **Next >**.

1. On the **Specify Name and Location** page of the **New Virtual Machine Wizard**, specify the following settings and select **Next >**:

    | Setting | Value | 
    | --- | --- |
    | Name | **az30312a-vm1** | 
    | Store the virtual machine in a different location | selected | 
    | Location | **F:\VMs** |

    >**Note**: Make sure to create the **F:\VMs** folder.

1. On the **Specify Generation** page of the **New Virtual Machine Wizard**, ensure that the **Generation 1** option is selected and select **Next >**:

1. On the **Assign Memory** page of the **New Virtual Machine Wizard**, set **Startup memory** to **2048** and select **Next >**.

1. On the **Configure Networking** page of the **New Virtual Machine Wizard**, in the **Connection** drop-down list select **NestedSwitch** and select **Next >**.

1. On the **Connect Virtual Hard Disk** page of the **New Virtual Machine Wizard**, select the option **Use an existing virtual hard disk**, set location to the VHD file you downloaded to the **F:\VHDs** folder, and select **Next >**.

1. On the **Summary** page of the **New Virtual Machine Wizard**, select **Finish**.

1. In the **Hyper-V Manager** console, select the newly created virtual machine and select **Start**. 

1. In the **Hyper-V Manager** console, verify that the virtual machine is running and select **Connect**. 

1. In the Virtual Machine Connection window to **az30312a-vm1**, on the **Hi there** page, select **Next**. 

1. In the Virtual Machine Connection window to **az30312a-vm1**, on the **License terms** page, select **Accept**. 

1. In the Virtual Machine Connection window to **az30312a-vm1**, on the **Customize settings** page, set the password of the built-in Administrator account to **Pa55w.rd1234** and select **Finish**. 

1. In the Virtual Machine Connection window to **az30312a-vm1**, sign in by using the newly set password.

1. In the Virtual Machine Connection window to **az30312a-vm1**, start Windows PowerShell and, in the **Administrator: Windows PowerShell** window run the following to set the computer name. 

   ```powershell
   Rename-Computer -NewName 'az30312a-vm1' -Restart
   ```

### Exercise 1: Create and configure an Azure Site Recovery vault
  
The main tasks for this exercise are as follows:

1. Create an Azure Site Recovery vault

1. Configure the Azure Site Recovery vault


#### Task 1: Create an Azure Site Recovery vault

1. Within the Remote Desktop session to **az30312a-hv-vm**, start Microsoft Edge, navigate to the [Azure portal](https://portal.azure.com), and sign in by providing credentials of a user account with the Owner role in the subscription you will be using in this lab.

1. In the Azure portal, search for and select **Recovery Services vaults** and, on the **Recovery Services vaults** blade, select **+ Create**.

1. On the **Basics** tab of the **Create Recovery Services vault** blade, specify the following settings (leave others with their default values) and select **Review + create**:

    | Setting | Value | 
    | --- | --- |
    | Subscription | the name of the Azure subscription you are using in this lab |
    | Resource group | the name of a new resource group **az30312b-labRG** |
    | Vault name | **az30312b-rsvault** |
    | Location | the name of the Azure region different from the one into which you deployed the virtual machine earlier in this lab |

1. On the **Review + create** tab of the **Create Recovery Services vault** blade, select **Create**:

    >**Note**: By default, the default configuration for Storage Replication type is set to Geo-redundant (GRS) and Soft Delete is enabled. You will change these settings in the lab to simplify deprovisioning, but you should use them in your production environments.


#### Task 2: Configure the Azure Site Recovery vault

1. In the Azure portal, search for and select **Recovery Services vaults** and, on the **Recovery Services vaults** blade, select **az30312b-rsvault**.

1. On the **az30312b-rsvault** blade, select **Properties**. 

1. On the **az30312b-rsvault | Properties** blade, select the **Update** link under the **Backup Configuration** label.

1. On the **Backup Configuration** blade, set **Storage replication type** to **Locally-redundant**, select **Save** and close the **Backup Configuration** blade.

    >**Note**: Storage replication type cannot be changed once you start protecting items.

1. On the **az30312b-rsvault | Properties** blade, select the **Update** link under the **Security Settings** label.

1. On the **Security Settings** blade, set **Soft Delete** to **Disable**, select **Save** and close the **Security Settings** blade.


### Exercise 2: Implement Hyper-V protection by using Azure Site Recovery vault
  
The main tasks for this exercise are as follows:

1. Implement the target Azure environment

1. Implement protection of a Hyper-V virtual machine

1. Perform a failover of the Hyper-V virtual machine

1. Remove Azure resources deployed in the lab


#### Task 1: Implement the target Azure environment

1. In the Azure portal, search for and select **Virtual networks** and, on the **Virtual networks** blade, select **+ Create**.

1. On the **Basics** tab of the **Create virtual network** blade, specify the following settings (leave others with their default values) and select **Next: IP Addresses**:

    | Setting | Value |
    | --- | --- |
    | Subscription | the name of the Azure subscription you are using in this lab |
    | Resource group | the name of a new resource group **az30312c-labRG** |
    | Name | **az30312c-dr-vnet** |
    | Region | the name of the Azure region into which you deployed the Recovery Services vault in the previous exercise of this lab |

1. On the **IP addresses** tab of the **Create virtual network** blade, in the **IPv4 address space** text box, type **10.7.0.0/16** and select **+ Add subnet**.

1. On the **Add subnet** blade, specify the following settings (leave others with their default values) and select **Add**:

    | Setting | Value |
    | --- | --- |
    | Subnet name | **subnet0** |
    | Subnet address range | **10.7.0.0/24** |

1. Back on the **IP addresses** tab of the **Create virtual network** blade, select **Review + create**.

1. On the **Review + create** tab of the **Create virtual network** blade, select **Create**.

1. In the Azure portal, search for and select **Virtual networks** and, on the **Virtual networks** blade, select **+ Create**.

1. On the **Basics** tab of the **Create virtual network** blade, specify the following settings (leave others with their default values) and select **Next: IP Addresses**:

    | Setting | Value |
    | --- | --- |
    | Subscription | the name of the Azure subscription you are using in this lab |
    | Resource group | **az30312c-labRG** |
    | Name | **az30312c-test-vnet** |
    | Region | the name of the Azure region into which you deployed the Recovery Services vault in the previous exercise of this lab |

1. On the **IP addresses** tab of the **Create virtual network** blade, in the **IPv4 address space** text box, type **10.7.0.0/16** and select **+ Add subnet**.

1. On the **Add subnet** blade, specify the following settings (leave others with their default values) and select **Add**:

    | Setting | Value |
    | --- | --- |
    | Subnet name | **subnet0** |
    | Subnet address range | **10.7.0.0/24** |

1. Back on the **IP addresses** tab of the **Create virtual network** blade, select **Review + create**.

1. On the **Review + create** tab of the **Create virtual network** blade, select **Create**.

1. In the Azure portal, search for and select **Storage accounts** and, on the **Storage accounts** blade, select **+ Create**.

1. On the **Basics** tab of the **Create storage account** blade, specify the following settings (leave others with their default values):

    | Setting | Value | 
    | --- | --- |
    | Subscription | the name of the Azure subscription you are using in this lab |
    | Resource group | **az30312c-labRG** |
    | Storage account name | any globally unique name between 3 and 24 in length consisting of letters and digits |
    | Location | the name of the Azure region in which you created the virtual network earlier in this task |
    | Performance | **Standard** |
    | Account kind | **StorageV2 (general purpose v2)** |
    | Replication | **Locally redundant storage (LRS)** |

1. On the **Basics** tab of the **Create storage account** blade, select **Review + create**.

1. On the **Review + create** tab of the **Create storage account** blade, select **Create**.


#### Task 2: Implement protection of a Hyper-V virtual machine

1. Within the Remote Desktop session to **az30312a-hv-vm**, in the Azure portal, on the **az30312b-rsvault** blade, in the **Site Recovery**section, select **Getting started**.

1. On the **az30312b-rsvault \| Site Recovery** blade, in the **Hyper-V machines to Azure** section, select **1. Prepare infrastructure**. 

1. On the **Deployment planning** tab of the **Prepare infrastructure** blade, in the **Deployment planning completed?** drop-down list, select **Yes, I have done it** and select **Next**.

1. On the **Source settings** tab of the **Prepare infrastructure** blade, next to the **Are you Using System Center VMM to manage Hyper-V hosts** label, select the **No** option.

1. On the **Source settings** tab of the **Prepare infrastructure** blade, select the **Add Hyper-V site** link, on the **Create Hyper-V Site** blade, in the **Name** text box, type **az30312b Hyper-V site** and select **OK**:

1. On the **Source settings** tab of the **Prepare infrastructure** blade, select the **Add Hyper-V server** link. 

1. On the **Add Server** blade, select the **Download** link in step 3 of the procedure for adding on-premises Hyper-V hosts in order to download the Microsoft Azure Site Recovery Provider.

1. When the download completes, select **Open file** then select **Run** to launch **AzureSiteRecoveryProvider.exe**. This will start the **Azure Site Recovery Provider Setup (Hyper-V server)** wizard.

1. On the **Microsoft Update** page, select **Off** and select **Next**.

1. On the **Provider installation** page, select **Install**.

1. Switch to the Azure portal and, on the **Add Server** blade, select the **Download** button in step 4 of the procedure for registering on-premises Hyper-V hosts in order to download the vault registration key. If prompted, select **Save** to save the vault credentials file in the **Downloads** folder.

1. Switch to the **Provider installation** wizard window and select **Register**. This will start the **Microsoft Azure Site Recovery Registration Wizard**.

1. On the **Vault Settings** page of the **Microsoft Azure Site Recovery Registration Wizard**, select **Browse**, in the **Open** window, navigate to the **Downloads** folder, select the vault credentials file, and select **Open**.

1. Back on the **Vault Settings** page of the **Microsoft Azure Site Recovery Registration Wizard**, select **Next**.

1. On the **Proxy Settings** page of the **Microsoft Azure Site Recovery Registration Wizard**, accept the default settings and select **Next**.

1. On the **Registration** page of the **Microsoft Azure Site Recovery Registration Wizard**, select **Finish**.

1. Switch back to the Internet Explorer window displaying the Azure portal and refresh the page. When prompted, select **Leave the page** or **Reload** depending on the version of the browser.

1. Back on the **az30312b-rsvault | Site Recovery** blade, in the **Hyper-V machines to Azure** section, select **1. Prepare infrastructure**. 

1. On the **Deployment planning** tab of the **Prepare infrastructure** blade, in the **Deployment planning completed?** drop-down list, select **Yes, I have done it** and select **Next**.

1. On the **Source settings** tab of the **Prepare infrastructure** blade, next to the **Are you Using System Center VMM to manage Hyper-V hosts** label, select the **No** option.

1. Verify that the **Hyper-V site** and **Hyper-V servers** settings are set correctly and select **Next**. 

1. On the **Target settings** tab of the **Prepare infrastructure** blade, accept the default settings and select **Next**.

1. On the **Replication policy** tab of the **Prepare infrastructure** blade, select **Create new policy and associate**. 

1. On the **Create and associate policy** blade, specify the following settings (leave others with their default values) and select **OK**:

    | Setting | Value |
    | --- | --- |
    | Name | **az30312b replication policy** |
    | Copy frequency | **30 seconds** |

1. Back on the **Replication policy** tab of the **Prepare infrastructure** blade, wait until the site has been associated with the policy and select **Next**.

1. On the **Review** tab of the **Prepare infrastructure** blade, select **Prepare**.

1. On the **az30312b-rsvault \| Site Recovery** blade, in the **Hyper-V machines to Azure** section, select **2. Enable replication**. 

1. On the **Source environment** tab of the **Enable replication** blade, in the **Source location** drop-down list, select **az30312b Hyper-V site** and select **Next**.

1. On the **Target environment** tab of the **Enable replication** blade, specify the following settings (leave others with their default values) and select **Next**:

    | Setting | Value |
    | --- | --- |
    | Subscription | the name of the Azure subscription you are using in this lab |
    | Post-failover resource group | **az30312c-labRG** |
    | Post-failover deployment model | **Resource Manager** |
    | Storage account | the name of the storage account you created in the first task of this exercise |
    | Network | Configure now for selected machines |
    | Virtual network | **az30312c-dr-vnet** |
    | Subnet | **subnet0 (10.7.0.0/24)** |

1. On the **Virtual machine selection** tab of the **Enable replication** blade, select the **az30312a-vm1** checkbox and select **Next**.

1. On the **Replication settings** tab of the **Enable replication** blade, in the **Defaults** row and **OS type** column, select **Windows** from the drop-down list and select **Next**.

1. On the **Replication policy** tab of the **Enable replication** blade, accept the default settings and select **Next**.

1. On the **Review** tab of the **Enable replication** blade, select **Enable replication**.

#### Task 3: Review Azure VM replication settings

1. In the Azure portal, back on the **az30312b-rsvault \| Site Recovery** blade, in the vertical menu, select **Replicated items**. 

1. On the **az30312b-rsvault \| Replicated items** blade, ensure that there is an entry representing the **az30312a-vm1** virtual machine and verify that its **Replication Health** is listed as **Healthy** and that its **Status** is listed as either **Enabling protection**, **Waiting for first recovery point**, or displaying a current percentage of synchronization progress.

   > **Note**: You might need to wait a few minutes until the **az30312a-vm1** entry appears on the **az30312b-rsvault \| Replicated items** blade.

1. On the **az30312b-rsvault \| Replicated items** blade, select the **az30312a-vm1** entry.

1. On the **az30312a-vm1** replicated items blade, review the **Health and status**, **Failover readiness**, **Latest recovery points**, and **Infrastructure view** sections. Note the **Planned Failover**, **Failover** and **Test Failover** toolbar icons.

   > **Note**: Wait until the status changes to **Protected**. This might take additional 15 minutes. You will need to refresh the browser page for the status to be updated.

1. On the **az30312a-vm1** replicated items blade, select **Latest recovery points** and review **Latest crash-consistent** and **Latest app-consistent** recovery points. 

#### Task 4: Perform a failover of the Hyper-V virtual machine

1. Within the Remote Desktop session to **az30312a-hv-vm**, in the browser window displaying the Azure portal, on the **az30312a-vm1** replicated items blade, select **Test failover**. 

1. On the **Test failover** blade, specify the following settings (leave others with their default values) and select **OK**:

    | Setting | Value |
    | --- | --- |
    | Choose a recovery point | the default option | 
    | Azure virtual network | **az30312c-test-vnet** | 


1. In the Azure portal, navigate back to the **az30312b-rsvault** blade and select **Site Recovery jobs**. Wait until the status of the **Test failover** job is listed as **Successful**.

1. In the Azure portal, search for and select **Virtual machines** and, on the **Virtual machines** blade, note the entry representing the newly provisioned virtual machine **az30312a-vm1-test**.

1. In the Azure portal, navigate back to the on the **az30312a-vm1** replicated items blade and select **Cleanup test failover*.

1. On the **Test failover cleanup** blade, select the checkbox **Testing is complete. Delete test failover virtual machine(s)** and select **OK**.

1. Once the test failover cleanup job completes, refresh the browser page displaying the **az30312a-vm1** replicated items blade and note that you have the option to perform planned and unplanned failover.

1. On the **az30312a-vm1** replicated items blade, select **Planned failover**. 

1. On the **Planned failover** blade, note that the failover direction settings are already set and not modifiable. 

1. Close the **Planned failover** blade and, on the **az30312a-vm1** replicated items blade, select **Failover**. 

1. On the **Failover** blade, note the available options geared towards minimizing potential data loss. 

1. Close the **Failover** blade.


#### Task 5: Remove Azure resources deployed in the lab

1. Within the Remote Desktop session to **az30312a-hv-vm**, in the browser window displaying the Azure portal, start a PowerShell session within the Cloud Shell pane.

1. From the Cloud Shell pane, run the following to list the resource group you created in this exercise:

   ```powershell
   Get-AzResourceGroup -Name 'az30312*'
   ```

    > **Note**: Verify that the output contains only the resource group you created in this lab. This group will be deleted in this task.

1. From the Cloud Shell pane, run the following to delete the resource group you created in this lab

   ```powershell
   Get-AzResourceGroup -Name 'az30312*' | Remove-AzResourceGroup -Force -AsJob
   ```

1. Close the Cloud Shell pane.
