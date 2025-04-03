# Azure Load Balancer Update

## Create Lab Envinroment

In this first session, we'll create a Load Balancer Basic to migrate the version to Standard.

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
