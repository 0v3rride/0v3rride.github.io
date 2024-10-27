# Offender 365

## Background
I recently completed the MCRTP exam from Pwnedlabs. The course offered a lot of vaulable insight that I felt the SANS GCPN glossed over. During the course, there were a couple of areas in Azure/O365 that piqued my intrests, and wanted to circle back on after completing the exam.

* What makes continous access elevation tokens special?
* Is there a way to acquire a token via the OAuth flow API with all the necessary scopes to access storage accounts and key vaults—one access token to rule them all?
  * The token should have similar permissions or scopes to the one you get via interactive login using `Connect-AzAccount`.
    
Another question I kept asking myself is, can other services or scopes be leveraged in some way within an organization’s tenant?

### Defender 365
What about using the blue team's tools against them? Sure, Defender 365 and Microsoft Defender for Endpoint offer some possibilities. One aspect is research by Thomas Naunheim titled, [Abuse and Detection of M365D Live Response for privilege escalation on Control Plane (Tier0) assets](https://www.cloud-architekt.net/abuse-detection-live-response-tier0/), which outlines leveraging Defender's live response feature to execute code on endpoints.

The way Microsoft has laid out Microsoft Defender for Endpoint and Defender 365 is a little confusing. Live response can be accessed via the `https://api.securitycenter.microsoft.com` scope with the appropriate permissions. Defender 365 also outfits blue teams with threat hunting capabilities. This comes in the form of advanced threat hunting in the Defender 365 portal, which provides tables to query for threat hunting operations. This includes Exchange emails, Defender for Endpoint alerts and incidents, and even device network, file, and process events. KQL is the query language used to conjure up results. The API for [threat hunting use to live on the Microsoft Threat Protection API](https://learn.microsoft.com/en-us/defender-endpoint/api/run-advanced-query-api) - (`api.securitycenter.microsoft.com`) and required the `AdvancedHunting.Read.All` API persmission to be granted on the Entra registered app that would use it. Since then Microsoft has moved [threat hunting to the Graph API](https://learn.microsoft.com/en-us/graph/api/security-security-runhuntingquery?view=graph-rest-1.0&tabs=http). The registered Entra app requires the API permission `ThreatHunting.Read.All` to be granted.

Therefore, if you manage to grab an access token with the `ThreatHunting.Read.All` (Microsoft Graph Security API) or `AdvancedHunting.Read.All` (Microsoft Threat Protection) API permission/role or the credentials of an entra user assigned with the security reader, security operator or security administrator entra role. Then you're presented with some interesting capabilities. Let me elaborate a little more. Advanced threat hunting offers two interesting tables/schemas to pull information about tenant resources via [exposure graph](https://learn.microsoft.com/en-us/security-exposure-management/query-enterprise-exposure-graph). Think of the exposure graph as Microsoft's version of Bloodhound/Azurehound. However, more specifically were interested in the [ExposureGraphNodes](https://learn.microsoft.com/en-us/defender-xdr/advanced-hunting-exposuregraphnodes-table) and [ExposureGraphEdges](https://learn.microsoft.com/en-us/defender-xdr/advanced-hunting-exposuregraphedges-table) tables with in advanced hunting. Essentially, the ExposureGraphNode table allows you to enumerate virtually every node (user, serviceprincpal, managed devices, etc.) within the organization's on-prem infrastructure and within the tenant. The best part is that the principal that has access to the advanced threat hunting __DOES NOT__ need to be explicitly assigned to any subscriptions, resources or have any RBAC role assignments based on at least one case I've seen in a production tenant.

### TLDR: If you get access to a principal who has the security reader, security operator or security admin entra role assigned to it, or if you get an access token with the above-mentioned API permissions/roles. Then you can see all of the resources deployed in the tenant using the ExposureGraphNode in advanced threat hunting. These resources include sites, function apps, databases and their respective tables, storage accounts and their respective containers, key vaults, VMs, VNets, public IP addresses and more.

Here's a basic query that looks for all possible Azure resources within the tenant:
```
ExposureGraphEdges
| where TargetNodeLabel matches regex "microsoft\\.*"
```

You could break this down even more and see what roles or permissions a source node has on a target node:
```
ExposureGraphEdges
| where SourceNodeLabel has_any ("user", "group", "role", "service", "managed") and TargetNodeLabel matches regex "microsoft\\.*"
| mv-expand EdgeProperties
| extend data = EdgeProperties.rawData
| extend permissions = data.permissions
| extend roles = permissions.roles
| mv-expand roles
| extend roleName = roles.name
| project SourceNodeLabel, SourceNodeName, roleName, TargetNodeLabel, TargetNodeName
| order by SourceNodeName asc
```

This is great and all, but there's still not a convienient way to interact with Defender 365 using APIs other than registering a app within Entra. 

So, time to use trusty old burp suite to analyze some requests. When logged into the Defender 365 portal as a user with the security reader role. I discovered that everytime a query was executed the request would be sent to the following URL, `https://security.microsoft.com/apiproxy/mtp/huntingService/queryExecutor?useFanOut=false`. 
Doing a quick Google search the [following article from Falcon Force](https://medium.com/falconforce/microsoft-defender-for-endpoint-internals-0x04-timeline-3f01282839e4), described this as an API proxy that Microsoft has put in place to prevent interaction in this manner. It is also mentioned that the folks at Falcon Force were able to write a Python script that works around the API proxy. This specific python script isn't shared though. 

So, I went back to playing around with Burp Suite. Upon replaying the request multiple times and removing various elements, I discovered that the bare minimum needed to make the request work is the following:

### Request Headers
* Cookie: sccauth=...
* X-Xsrf-Token: ...
* Content-Type: application/json

### Post Request JSON Data
`{
   "QueryText": "KQL query here",
   "StartTime": "YYYY-mm-ddTHH:MM:sssZ", //can be None/Null
   "EndTime": "YYYY-mm-ddTHH:MM:sssZ", //can be None/Null
   "MaxRecordCount": 10 //can be None/Null
}`

Then I found a [follow up article from Falcon Force](https://falconforce.nl/microsoft-defender-for-endpoint-internals-0x05-telemetry-for-sensitive-actions/), which goes into more detail as to how one could create a work around for the API proxy Microsoft has in place. Olaf developed a tool called [DefenderHarvester] (https://github.com/olafhartong/DefenderHarvester/tree/30cfb7558471d2bbadfd279b9c66cb889e9fdfbb) to demonstrate this. The tool makes use of what Olaf calls MDE service APIs. You can view these service API URLs using Burp Suite and examining the response headers. The response header for the specific service you're interested in will be shown in the `X-Source-Url` value. In my case, I am interested in using the threat hunting API. Remember that `https://security.microsoft.com/apiproxy/mtp/huntingService/queryExecutor?useFanOut=false` is the API proxy. Upon reviewing the response headers, I found that the 'real' API the request was being forwarded to was `https://m365d-hunting-api-prd-weu.securitycenter.windows.com/api/ine/huntingservice/queryExecutor?useFanOut=false`. Now, instead of having to use the `sccauth` cookie and `x-xsrf-token` header to make a request, we can simply just grab an access token for the scope `https://securitycenter.microsoft.com/mtp/.default`. This was found by [reading the source code of DefenderHarvester](https://github.com/olafhartong/DefenderHarvester/blob/30cfb7558471d2bbadfd279b9c66cb889e9fdfbb/main.go#L85) and implementing this into my tool. Everything about the request is the same, but this time all that is needed for the headers is the authorization bearer access token that you can acquire in a multitude of different ways. There seems to be caveat though under certain unknown circumstances. In my case when requesting the access token it forced me to do it from a managed device via interactive login with the Edge browser. Any other browser I tried gave me an error. I also tried a couple of different methods without using interactive login, and they all resulted in errors about the device not being managed. In my case this issue only occured for a user in a specific tenant that I was testing. So, this may be related to a policy that is in place on the tenant. I did some additional tests on two other users who were part of different tenants and did not recieve any issues about the device being unmanaged. I'm unsure what the exact cause was in my case, but I'll have to look more into it.

Now we have two ways to access the APIs within Defender 365. One being logging into to security.microsoft.com via browser and pulling the needed header values with inspect element and then sending the post request with the desired values for the parameters. The second using a token acquired from `securitycenter.microsoft.com/mtp` and using it with requests to appropriate MDE service APIs. I should note that the `securitycenter.microsoft.com` is an older domain and has been 'replaced' by the `api.security.microsoft.com` and  `security.microsoft.com` domains. So, this method could stop working at any moment.

What else can we try using to get access to Defender 365 API endpoints? Well, I tried mimicking the interactive login flow with python request sessions to no avail. Feel free to give this a shot yourself and please let me know if you get this working as it could be a solid replacement in the future.

```
import requests
import webbrowser

url = "https://security.microsoft.com/"
session = requests.Session()

# Get cookies (openidconnect.nonce, etc.)
response = session.get(url, allow_redirects=False)

# Format cookie values for Cookie header
cookies = session.cookies.get_dict()
cookieString = "; ".join([f"{k}={v}" for k, v in cookies.items()])

# Location has the full url and get params needed to make the auth code oauth flow request
location = response.headers.get("Location")

params = location.split("?")[1].split("&")

# Remove unecessary params
params.pop(4)
params.pop(6)

# Storing these just in case they are needed later
openidconnect_properties = params[3]
nonce = params[4]

# Join the params
params = "&".join(params)

webbrowser.open(f"https://login.microsoftonline.com/common/oauth2/authorize?{params}")

authcode_url = input("Input entire url with authcode here (url in browser navbar on blank page): ")
post_params = authcode_url.split("#")[1].split("&")
code = id_token = state = session_state = correlation_id = None

code = None
id_token = None
state = None
session_state = None
correlation_id = None

# Take values from initial url auth code flow and place them into dict for post request
for p in post_params:
    key, value = p.split("=")
    if key == "code":
        code = value
    elif key == "id_token":
        id_token = value
    elif key == "state":
        state = value
    elif key == "session_state":
        session_state = value
    elif key == "correlation_id":
        correlation_id = value

body = {
    "code": code,
    "id_token": id_token,
    "state": state,
    "session_state": session_state,
    "correlation_id": correlation_id
}

# Add cookies from initial get request in the session and add some other headers
headers = {
    "Cookie": cookieString,
    'Content-Type': 'application/x-www-form-urlencoded',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/128.0.6613.120 Safari/537.36',
    'Origin': 'https://login.microsoftonline.com',
    'Referer': 'https://login.microsoftonline.com/'
}

print("--------------------------------------------------------------------")
print()
print(f"https://login.microsoftonline.com/common/oauth2/authorize?{params}")
print()
print(cookies)
print()
print(body)
print()
print(headers)

response = session.post(url, data=body, headers=headers, proxies={"http": "http://127.0.0.1:8080", "https": "http://127.0.0.1:8080"}, verify=False)

print(response.cookies.get_dict())

# Despite everything looking like it is setup correctly following the requests in burp. The response is a 440 - signin reason timeout
```

When doing some troubleshooting to the script above [I found that ROADtools uses selenium to manage interactive logins](https://github.com/search?q=%2Fopenidconnect%5C.nonce%2F&type=code). I don't really like using selenium for things unless I absolutely have to. I figured if it's good enough for ROADtools then it wouldn't hurt to use it in this application as well. So, with no other choice. My last idea was to use `selenium-wire` to automate the interactive login flow to security.microsoft.com and obtain the `sccauth` cookie value and `X-Xsrf-token` response header value. This method is kind of shotty depending on whether the organization's tenant is managed or federated. 

I've added both methods to run threat hunting queries in Habu2:
* Selenium (`SeleniumRunQuery`)
* MDE service API for the threat hunting service (`MTPRunQuery`)

## Resources
* https://learn.microsoft.com/en-us/graph/api/security-security-runhuntingquery?view=graph-rest-1.0&tabs=http
* https://www.reddit.com/r/DefenderATP/comments/w9cj8y/pushing_detection_rules_programmatically/
* https://github.com/fooop/DefenderDetectionSync
* https://medium.com/falconforce/microsoft-defender-for-endpoint-internals-0x04-timeline-3f01282839e4
* https://medium.com/falconforce/microsoft-defender-for-endpoint-internals-0x05-telemetry-for-sensitive-actions-1b90439f5c25
* https://github.com/fahri314/365-Defender-Rule-Activator
* https://github.com/search?q=%2Fapiproxy%5C%2Fmtp%5C%2Fhunting%2F&type=code
* https://github.com/search?q=%2Fsecurity%5C.microsoft%5C.com%5C%2Fapiproxy%5C%2Fmtp%2F&type=code
* https://security.microsoft.com/v2/advanced-hunting?tid=tenantidoralias