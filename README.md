# Azure Load Balancer Update

## Create Lab Envinroment

In this first stage, we will continue with the creation of the Lab to test the migration of LB from Basic to Standard.

```powershell
# ==============================================
# 1. Variables and Context
# ==============================================
$tenantId = "ba249fd3-0485-4c68-8488-b3e345d63447"
$subscriptionName = "MCAPS"
$resourceGroupName = "rg-lab-loadbalancer"
$location = "North Europe"

# Load Balancer Components
$publicIpName = "pip-lb-basic"
$loadBalancerName = "lb-basic-web"
$frontendConfigName = "frontendConfig"
$healthProbeName = "probe-http"
$backendPoolName = "bepool-web"
$loadBalancingRuleName = "lbrule-http"
$frontendPort = 80
$backendPort = 80
$probePort = 80
$probeInterval = 5
$probeCount = 2

# VM and Network
$netPrefix = "lab"
$vnetName = "$netPrefix-vnet"
$subnetName = "$netPrefix-subnet"
$vnetPrefix = "192.168.0.0/24"
$subnetPrefix = "192.168.0.0/24"
$nicName = "nic-web-01"
$vmName = "vm-web-01"
$vmSize = "Standard_B2s"
$adminUsername = "azureuser"
$adminPassword = ConvertTo-SecureString "MyS3cure!Pwd2025" -AsPlainText -Force  # ✅ senha válida e aceita
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
## Enable IIS and configure the test page

After creating the lab environment, let's enable IIS and set up a test page for this webserver.

Navigate to Operations "Run Command" 
![image](https://github.com/user-attachments/assets/4656d176-8ea9-41d6-823b-b574161ec2eb)

Then run "RunPowerShellScript"
![image](https://github.com/user-attachments/assets/f8bd9d24-b554-4832-91d1-eace5a0cee53)

```pooweshell
# Install IIS with Management Tools
Add-WindowsFeature Web-Server -IncludeManagementTools

# Remove the iisstart page
Remove-Item C:\inetpub\wwwroot\iisstart.htm

# Configure the Default pages
Add-Content -Path "C:\inetpub\wwwroot\Default.htm" -Value "web-lab-loadbalancer -- $($env:computername)"
```

![image](https://github.com/user-attachments/assets/60aee825-137c-46de-b3fa-484e2ea7bfb4)

Wait for the process to finish - this usually takes a while

![image](https://github.com/user-attachments/assets/fb6fb033-0bd8-4fea-8aa0-518c93d69ef7)

Take LB's public IP address and run a page test. The expected result is this.

![image](https://github.com/user-attachments/assets/3d6c4781-feaf-49b6-9b44-f855b7939e10)

## Upgrade a basic load balancer with PowerShell

The upgrade process has been taken from the following reference documentation, for more details and information see the link: https://github.com/Azure/AzLoadBalancerMigration/tree/main/AzureBasicLoadBalancerUpgrade#upgrade-overview

###Important Considerations When Migrating Load Balancer (Basic ➜ Standard)

**1. Connectivity Interruption During Migration**
- The module reassociates VM NICs to the new Standard Load Balancer backend pool.
- This can cause a brief network interruption, especially for live traffic workloads.
- For production environments, schedule a maintenance window to avoid impact.

**2. Automatic Backup**
- The -RecoveryBackupPath flag generates a JSON file containing the Basic Load Balancer configuration.
- The backup does not include external resources such as NSGs, VMs, or old static IPs.
- Keep the backup in case you need to perform a manual rollback using Restore-AzLoadBalancerConfig.

**3. New Public IP**
- Standard Load Balancers cannot use the same public IP as the Basic SKU.
- The Public IP will be upgraded to Standard

**4. Security Rules / NSG**
- Make sure your NSG (Network Security Group) on the subnet or NIC allows traffic to the new configuration (e.g. port 80).
- The new IP may be blocked by existing deny rules if you're using source/destination filters.

**5. Old Resource Cleanup**
- The module does not delete the Basic Load Balancer or its associated Public IP.
- You should manually remove them after validating the new Standard LB:

**6. Required Permissions**
- **"Network Contributor"** Create/modify networking resources (LB, NICs, Public IPs) 
- **"Virtual Machine Contributor"** Read and modify VM and NIC configurations 
- **"Reader"** Create and associate backend pools 

**7. NAT Rules and Outbound Rules**
- The module does not automatically migrate NAT rules or outbound rules.
- If your Basic Load Balancer has NAT rules (e.g., RDP or SSH), you will need to manually recreate them on the new Standard LB.

**8. Unsupported Scenarios**
- Basic Load Balancers with IPV6 frontend IP configurations
- Basic Load Balancers with a Virtual Machine Scale Set backend pool member where one or more Virtual Machine Scale Set instances have ProtectFromScaleSetActions Instance Protection policies enabled
- Migrating a Basic Load Balancer to an existing Standard Load Balancer

**9. Manual Rollback (if needed)**
- If something goes wrong, you can restore the Basic LB configuration from the backup



##Module Installation

**Install the module from PowerShell gallery**

```powershell
Install-Module -Name AzureBasicLoadBalancerUpgrade -Scope CurrentUser -Repository PSGallery -Force
```

**Validate a scenario**

```powershell
Start-AzBasicLoadBalancerUpgrade -ResourceGroupName $resourceGroupName -BasicLoadBalancerName $loadBalancerName -validateScenarioOnly
```

**Upgrade with alternate backup path**

```powershell
Start-AzBasicLoadBalancerUpgrade -ResourceGroupName <loadBalancerRGName> -BasicLoadBalancerName $loadBalancerName -StandardLoadBalancerName "$($loadBalancerName)-std" -RecoveryBackupPath C:\temp\BasicLBRecovery
```

