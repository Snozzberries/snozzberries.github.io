# Azure Arc Managed Identities

[Previously](https://snozzberries.github.io/2022/05/15/AzureManagedIdentityPowerShell.html) I wrote up how to work with the Azure Instance Metadata Service (IMDS) with Azure Virtual Machines to utilize Azure Managed Identities.

Managed identities can provide many benefits and can be even considered a modern replacement for [group managed service accounts (gMSA)](https://learn.microsoft.com/en-us/windows-server/security/group-managed-service-accounts/group-managed-service-accounts-overview) in Active Directory (AD) Domain Services (DS). Especially as you have growing needs to work with process automation of your cloud services.

Now you can see an example of using this concept with four modern enhancements.

* You can use Managed Identities anywhere with Azure Arc
* You can use them:
  * with the Az PowerShell modules
  * to obtain bearer tokens
  * with the Microsoft Graph PowerShell modules

## Prerequisites

You will need six pieces to make this come together:

* Familiarity with Azure and PowerShell
* Entra ID Tenant
* Azure Subscription
* Azure Resource Group
* A Windows Server instance
* PowerShell 7.*

> The [Azure Arc Connected Machine Agent supports operating system environments](https://learn.microsoft.com/en-us/azure/azure-arc/servers/prerequisites#supported-operating-systems) other than Windows Server as well.

## Setup Azure Arc

Your experience setting up Azure Arc should be painless.

### Identify an Azure Resource Group

Within your Azure Subscription, identify an existing Azure Resource Group (RG) or create a new RG that your Azure Arc machines will reside in. Make sure you note the region you use as well.

![image](https://github.com/Snozzberries/snozzberries.github.io/assets/431932/99bf56fc-1170-4267-85ad-955045d29d2d)

Azure Resource Group “Arc” within the Azure Portal.

### Setup Arc

Once you have an RG the machine will reside in. Open PowerShell on your instance and install the necessary modules, connect to Azure, and then connect the machine. You can see these commands shown below. You can replace `Arc` with your RG name, `$env:COMPUTERNAME` with a resource name you prefer, or `eastus` with another Azure Region.

> You can use `-UseDeviceAuthentication` to do a device code flow authentication in case you are working in a headless environment like Windows Server Core.

```powershell
Install-Module Az.ConnectedMachine, Az.Accounts
Connect-AzAccount -UseDeviceAuthentication
Connect-AzConnectedMachine  -ResourceGroupName Arc -Name $env:COMPUTERNAME -Location eastus
```

> Depending on how you want to use your identity, it is worth taking note that the `Connect-AzConnectedMachine` cmdlet creates the `IDENTITY_ENDPOINT` machine environment variable. Meaning you may need to start a new PowerShell session, reboot, or manually set the environment variable equal to `[System.Environment]::GetEnvironmentVariable("IDENTITY_ENDPOINT","Machine")` for the following sections to work.

### Use your machine's identity

Now you can just append the `-Identity` property to your connect cmdlets and you will authenticate with your managed identity.

```powershell
Connect-AzAccount -Identity
```

You will find that your managed identity does not have any subscription when it connects though.

```powershell
Account   SubscriptionName TenantId                             Environment
-------   ---------------- --------                             -----------
MSI@<ID>                   <GUID>                               AzureCloud
```

You can confirm this by getting your current context.

```powershell
Get-AzContext|Format-List
```

```powershell
Name               : <GUID> - MSI@<ID>
Account            : MSI@<ID>
Environment        : AzureCloud
Subscription       :
Tenant             : <GUID>
TokenCache         :
VersionProfile     :
ExtendedProperties : {}
```

Once you add your managed identity to an Azure Role Assignment you can connect again and see your updated context includes the subscription.

> If you make role assignments to multiple subscriptions ensure you set the appropriate subscription you want the Az cmdlets to use with `Set-AzContext`.

### Use your machine's identity token

You can also request a bearer token from the Azure Arc IMDS that you can then use with direct API calls. [Microsoft's documentation](https://learn.microsoft.com/en-us/azure/azure-arc/servers/managed-identity-authentication#acquiring-an-access-token-using-rest-api) provides a Windows PowerShell example. The following is a PowerShell Core example.

```powershell
$apiVersion = "2020-06-01"
$resource = "https://management.azure.com/"
$endpoint = "{0}?resource={1}&api-version={2}" -f $env:IDENTITY_ENDPOINT,$resource,$apiVersion
$secretFile = ""
try
{
    Invoke-WebRequest -Method GET -Uri $endpoint -Headers @{Metadata='True'} -UseBasicParsing
}
catch
{
    $wwwAuthHeader = $_.Exception.Response.Headers|?{$_.Key -eq "WWW-Authenticate"}|% value
    if ($wwwAuthHeader -match "Basic realm=.+")
    {
        $secretFile = ($wwwAuthHeader -split "Basic realm=")[1]
    }
}
Write-Host "Secret file path: " $secretFile`n
$secret = cat -Raw $secretFile
$response = Invoke-WebRequest -Method GET -Uri $endpoint -Headers @{Metadata='True'; Authorization="Basic $secret"} -UseBasicParsing
if ($response)
{
    $token = (ConvertFrom-Json -InputObject $response.Content).access_token
    Write-Host "Access token: " $token
}
```

### Use your machine's identity with the Graph API

Another use case that comes up is working with the Microsoft Graph API. The `Connect-MgGraph` cmdlet also supports managed identity authentication though. As simple as before, you can use your Azure Arc Connected Machine's Managed Identity.

```powershell
Install-Module Microsoft.Graph.Authentication
Connect-MgGraph -Identity
Get-MgContext
```

## Security thoughts

As you look to use managed identities to simplify your operational environment, please be mindful that often the log events for any access will only show the managed identity service principal as the client. Meaning if you allow interactive access to an instance with a managed identity, you need to correlate the local systems security logs with the API authentication logs to see who the real identity is interacting with the API as the managed identity.

The Cybersecurity and Infrastructure Security Agency (CISA) in a joint effort with other national defense agencies around the world provided [guidance on PowerShell configurations](https://www.cisa.gov/news-events/alerts/2022/06/22/keeping-powershell-measures-use-and-embrace).
