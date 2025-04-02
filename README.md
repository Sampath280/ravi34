```c#
using System;
using System.Linq;
using System.Threading.Tasks;
using Azure;
using Azure.Identity;
using Azure.ResourceManager;
using Azure.ResourceManager.AppService;
using Azure.ResourceManager.AppService.Models;
using Azure.ResourceManager.Dns;
using Azure.ResourceManager.Dns.Models;
using Azure.ResourceManager.Resources;

public class AzureService
{
    private readonly ArmClient _client;
    private readonly SubscriptionResource _subscription;
    private readonly ResourceGroupResource _resourceGroup;

    public AzureService(string resourceGroupName)
    {
        _client = new ArmClient(new DefaultAzureCredential());
        _subscription = _client.GetDefaultSubscription();
        _resourceGroup = _subscription.GetResourceGroup(resourceGroupName);
    }

    public async Task AddSubdomainAsync(string appServiceName, string subdomain, string domainName)
    {
        var appService = await GetAppServiceAsync(appServiceName);

        string subdomainName = subdomain.Replace($".{domainName}", "");
        string txtRecordName = $"asuid.{subdomainName}";

        // Fetch the verification ID from App Service
        string verificationId = appService.Data.CustomDomainVerificationId;

        // Create TXT Record for domain verification
        await AddTxtRecordAsync(domainName, txtRecordName, verificationId);

        // Create the hostname binding
        var hostNameBindingData = new HostNameBindingData
        {
            SslState = HostNameBindingSslState.SniEnabled,
            HostNameType = AppServiceHostNameType.Verified,  // Fixed type
            AzureResourceType = AppServiceResourceType.Website // Fixed type
        };

        await appService.GetSiteHostNameBindings().CreateOrUpdateAsync(WaitUntil.Completed, subdomain, hostNameBindingData);

        // Add CNAME Record pointing to App Service
        await AddCNameRecordAsync(domainName, subdomainName, $"{appServiceName}.azurewebsites.net");
    }

    public async Task AddCNameRecordAsync(string dnsZoneName, string recordSetName, string alias)
    {
        var dnsZone = await GetDnsZoneAsync(dnsZoneName);
        var cnameRecordSetData = new DnsCnameRecordData
        {
            TtlInSeconds = 3600,
            Cname = alias
        };

        await dnsZone.GetDnsCnameRecords().CreateOrUpdateAsync(WaitUntil.Completed, recordSetName, cnameRecordSetData);
    }

    public async Task AddTxtRecordAsync(string dnsZoneName, string recordSetName, string value)
    {
        var dnsZone = await GetDnsZoneAsync(dnsZoneName);
        var txtRecordSetData = new DnsTxtRecordData
        {
            TtlInSeconds = 3600,
            DnsTxtRecords = { new DnsTxtRecordInfo { Values = { value } } }
        };

        await dnsZone.GetDnsTxtRecords().CreateOrUpdateAsync(WaitUntil.Completed, recordSetName, txtRecordSetData);
    }

    private async Task<DnsZoneResource> GetDnsZoneAsync(string dnsZoneName)
    {
        return await _resourceGroup.GetDnsZoneAsync(dnsZoneName);
    }

    public async Task<WebSiteResource> GetAppServiceAsync(string appServiceName)
    {
        return await _resourceGroup.GetWebSiteAsync(appServiceName);
    }
}

```
