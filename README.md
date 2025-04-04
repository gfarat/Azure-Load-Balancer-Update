# Azure Load Balancer Upgrade Lab

This guide walks through the process of creating a lab environment in Azure to test the migration of a **Basic Load Balancer** to a **Standard Load Balancer** using the PowerShell module [`AzureBasicLoadBalancerUpgrade`](https://github.com/Azure/AzLoadBalancerMigration).

---

## Table of Contents

- [Create Lab Environment](#create-lab-environment)
- [Enable IIS and Configure the Test Page](#enable-iis-and-configure-the-test-page)
- [Important Considerations](#important-considerations-when-migrating-load-balancer-basic--standard)
- [Module Installation](#module-installation)
- [Scenario Validation](#validate-a-scenario)
- [Run the Upgrade](#upgrade-with-alternate-backup-path)
- [Validate the Upgrade](#validate-completed-migration)
- [Failure Recovery Procedure](#the-basic-failure-recovery-procedure-is)
- [Frequently Asked Questions (FAQ)](#frequently-asked-questions-faq)

---

## Create Lab Environment

> Full PowerShell script to deploy:
> - Basic Load Balancer
> - Public IP
> - Virtual Network/Subnet
> - Windows VM with NIC associated to LB backend pool

```powershell
# ==============================================
# 1. Variables and Context
# ==============================================
$tenantId = "Tenant ID"
$subscriptionName = "Subscription Name"
$resourceGroupName = "rg-lab-loadbalancer-01"
$location = "North Europe"

# Load Balancer Components
$publicIpName = "pip-lb-basic-01"
$loadBalancerName = "lb-basic-web-01"
$stdloadBalancerName = "lb-standard-web-01"
$frontendConfigName = "frontendConfig-01"
$healthProbeName = "probe-http-01"
$backendPoolName = "bepool-web-01"
$loadBalancingRuleName = "lbrule-http-01"
$frontendPort = 80
$backendPort = 80
$probePort = 80
$probeInterval = 5
$probeCount = 2

# VM and Network
$netPrefix = "lab-01"
$vnetName = "$netPrefix-vnet"
$subnetName = "$netPrefix-subnet"
$vnetPrefix = "192.168.0.0/24"
$subnetPrefix = "192.168.0.0/24"
$nicName = "nic-web"
$vmName = "vm-web"
$vmSize = "Standard_B2s"
$adminUsername = "azureuser"
$adminPassword = ConvertTo-SecureString "MyS3cure!Pwd2025" -AsPlainText -Force
$imagePublisher = "MicrosoftWindowsServer"
$imageOffer = "WindowsServer"
$imageSku = "2022-datacenter"

# ==============================================
# 2. Connect to Azure and Set Context
# ==============================================
Connect-AzAccount -Tenant $tenantId
Set-AzContext -Subscription $subscriptionName

# ==============================================
# 3. Create Resource Group
# ==============================================
New-AzResourceGroup -Name $resourceGroupName -Location $location

# ==============================================
# 4. Create Load Balancer
# ==============================================
$publicIP = New-AzPublicIpAddress -ResourceGroupName $resourceGroupName `
  -Name $publicIpName -Location $location -AllocationMethod Dynamic -Sku Basic

$frontendIP = New-AzLoadBalancerFrontendIpConfig -Name $frontendConfigName -PublicIpAddress $publicIP

$probe = New-AzLoadBalancerProbeConfig -Name $healthProbeName `
  -Protocol Tcp -Port $probePort -IntervalInSeconds $probeInterval -ProbeCount $probeCount

$backendPool = New-AzLoadBalancerBackendAddressPoolConfig -Name $backendPoolName

$rule = New-AzLoadBalancerRuleConfig -Name $loadBalancingRuleName `
  -FrontendIpConfiguration $frontendIP -BackendAddressPool $backendPool -Probe $probe `
  -Protocol Tcp -FrontendPort $frontendPort -BackendPort $backendPort

New-AzLoadBalancer -ResourceGroupName $resourceGroupName -Name $loadBalancerName -Location $location `
  -FrontendIpConfiguration $frontendIP -BackendAddressPool $backendPool -LoadBalancingRule $rule `
  -Probe $probe -Sku Basic

# ==============================================
# 5. Create Virtual Network and Subnet
# ==============================================
$subnetConfig = New-AzVirtualNetworkSubnetConfig -Name $subnetName -AddressPrefix $subnetPrefix

$vnet = New-AzVirtualNetwork -Name $vnetName -ResourceGroupName $resourceGroupName `
  -Location $location -AddressPrefix $vnetPrefix -Subnet $subnetConfig

# ==============================================
# 6. Create NIC with Backend Pool Association
# ==============================================
$loadBalancer = Get-AzLoadBalancer -ResourceGroupName $resourceGroupName -Name $loadBalancerName
$backendPool = $loadBalancer.BackendAddressPools | Where-Object { $_.Name -eq $backendPoolName }

$ipConfig = New-AzNetworkInterfaceIpConfig -Name "ipconfig1" `
  -SubnetId $vnet.Subnets[0].Id `
  -LoadBalancerBackendAddressPoolId $backendPool.Id

$nic = New-AzNetworkInterface -Name $nicName -ResourceGroupName $resourceGroupName `
  -Location $location -IpConfiguration $ipConfig

# ==============================================
# 7. Create VM
# ==============================================
$cred = New-Object PSCredential($adminUsername, $adminPassword)

$vmConfig = New-AzVMConfig -VMName $vmName -VMSize $vmSize |
  Set-AzVMOperatingSystem -Windows -ComputerName $vmName -Credential $cred -ProvisionVMAgent -EnableAutoUpdate |
  Set-AzVMSourceImage -PublisherName $imagePublisher -Offer $imageOffer -Skus $imageSku -Version "latest" |
  Add-AzVMNetworkInterface -Id $nic.Id

New-AzVM -ResourceGroupName $resourceGroupName -Location $location -VM $vmConfig

```
## Enable IIS and Configure the Test Page

After provisioning, run the following in **Run Command** from Azure Portal:

Navigate to Operations "Run Command" 
![image](https://github.com/user-attachments/assets/4656d176-8ea9-41d6-823b-b574161ec2eb)

Then run "RunPowerShellScript"
![image](https://github.com/user-attachments/assets/f8bd9d24-b554-4832-91d1-eace5a0cee53)

```powershell
Add-WindowsFeature Web-Server -IncludeManagementTools
Remove-Item C:\inetpub\wwwroot\iisstart.htm
Add-Content -Path "C:\inetpub\wwwroot\Default.htm" -Value "web-lab-loadbalancer -- $($env:computername)"
```

![image](https://github.com/user-attachments/assets/60aee825-137c-46de-b3fa-484e2ea7bfb4)

Wait for the process to finish - this usually takes a while

![image](https://github.com/user-attachments/assets/fb6fb033-0bd8-4fea-8aa0-518c93d69ef7)

Take LB's public IP address and run a page test. The expected result is this.

![image](https://github.com/user-attachments/assets/3d6c4781-feaf-49b6-9b44-f855b7939e10)

## Upgrade a basic load balancer with PowerShell

For more details and information not described in this documentation see the link: https://github.com/Azure/AzLoadBalancerMigration/tree/main/AzureBasicLoadBalancerUpgrade#upgrade-overview

## Important Considerations When Migrating Load Balancer (Basic ➜ Standard)

- Brief **network interruption** may occur.
- Existing **NSG rules** must allow new Standard LB IP.
- Public IP is upgraded to **Standard SKU**.

**Required Permissions**
- **"Network Contributor"** Create/modify networking resources (LB, NICs, Public IPs) 
- **"Virtual Machine Contributor"** Read and modify VM and NIC configurations 
- **"Reader"** Create and associate backend pools
  
> ℹ️ See [FAQ](#frequently-asked-questions-faq) for more edge cases.

## Module Installation

**Install the module from PowerShell gallery**

```powershell
Install-Module -Name AzureBasicLoadBalancerUpgrade -Scope CurrentUser -Repository PSGallery -Force
```

## Validate a Scenario

```powershell
Start-AzBasicLoadBalancerUpgrade -ResourceGroupName $resourceGroupName -BasicLoadBalancerName $loadBalancerName -validateScenarioOnly
```

## Upgrade with Alternate Backup Path

If you run the process with the “-RecoveryBackupPath” function, make sure you have created the directory before running it, as it doesn't create it and can cause an error during the migration. In my example, the directory I created was “C:\Temp\BasicLBRecovery” and the BasicLBRecovery directory will contain the backup json.

```powershell
Start-AzBasicLoadBalancerUpgrade -ResourceGroupName $resourceGroupName -BasicLoadBalancerName $loadBalancerName -StandardLoadBalancerName "$stdloadBalancerName" -RecoveryBackupPath "C:\temp\BasicLBRecovery" -FollowLog
```

LB before running the migration process

![image](https://github.com/user-attachments/assets/cf4703d7-f189-44b0-b046-f16519ada0ac)

LB after running the migration process, note that the SKU has changed and the name has also been changed by the variable configured in powershell.

![image](https://github.com/user-attachments/assets/1c8a6b7c-940d-4d25-8b9c-97de52077b97)

## Validate Completed Migration

After finishing the process, validate that the backup file was created in the specified directory and validate a completed migration by passing the Basic Load Balancer state file backup and the Standard Load Balancer name

![image](https://github.com/user-attachments/assets/e5ada653-89bd-4f14-b700-304ac95acda1)

```powershell
Start-AzBasicLoadBalancerUpgrade -validateCompletedMigration -StandardLoadBalancerName "$stdloadBalancerName" -basicLoadBalancerStatePath "C:\Temp\BasicLBRecovery\State_LB_lb-basic-web-01_rg-lab-loadbalancer-01_20250404T1200566613.json"
```
If the command returns with no message, it means that the process was successfully executed.

![image](https://github.com/user-attachments/assets/88e21d8d-136d-4980-92e4-0f4dfd36716c)

Validation of the WEB application with the IP preserved after migration.

![image](https://github.com/user-attachments/assets/d389a013-c895-4b31-bf24-b5f809079f40)

## The Basic Failure Recovery Procedure Is:

1. Check the log file: `Start-AzBasicLoadBalancerUpgrade.log`
2. Remove the failed Standard LB.
3. Locate the Basic LB backup file.
4. Rerun the migration with:

```powershell
-FailedMigrationRetryFilePathLB <BasicLBBackupFilePath>
```

## Frequently Asked Questions (FAQ)

### Will this migration cause downtime?
Yes. The Basic LB is removed before the Standard LB is created.

### Will my frontend IP be preserved?
Yes. Public IPs are converted to static. Internal IPs are reassigned if available.

### How long does it take?
Usually a few minutes, depending on configuration complexity.

### What components are migrated?
- Rules, probes, NATs, backend pools, IPs, tags, NSGs, etc.

### What if my VMs belong to multiple LBs?
Migrate all at once using `-MultiLBConfig`.

### How do I validate migration success?
Use `-validateCompletedMigration` with the backup file path.

### What if the upgrade fails mid-migration?
Fix the issue and rerun using `-FailedMigrationRetryFilePathLB`.

## License

This lab setup is based on the official [AzureBasicLoadBalancerUpgrade](https://github.com/Azure/AzLoadBalancerMigration) project and extended with additional lab scripting and validation steps.
