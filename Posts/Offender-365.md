# Offender 365

## Background
I recently completed the MCRTP exam from Pwnedlabs. Finally an offensive Azure course on the same tier as those offered by Offensive Security. The course itself included a lot of information that I feel like the SANS GCPN glossed over.
During the course there were a couple of areas in Azure/O365 that piqued my intrests and wanted to circle back on after completing the exam.

* What makes continous access elevation tokens special?
* Is there a way to acquire a token via the OAuth flow API with all the necessary scopes to access storage accounts, key vaults with one access token to rule them all?
  * The token should have similar permissions or scopes to the one you get via interactive login using Connect-AzAccount.
    
I'm still digging for answers in the points above, but there's other questions I've been able to find the answer to myself. One I keep asking myself is, can other services or scopes be used as attack paths in an orginization's tenant? Yep!

I remember Ian, our instructor and subject matter expert for the MCRTP, mentioning during one of the sessions that he was 
looking into Intune as an attack path. Having worked with Intune a little. I know with the appropriate permissions you can execute code remotely (Powershell) on a managed device of your choice. Which in theory should allow you to move laterally from the orginazation's cloud enviroment to on-prem.

### Defender 365
What about using the blue team's tools against them? Sure, Defender 365 & Microsoft Defender for Endpoint offer some possibilities. One aspect is research by Thomas Naunheim titled, [Abuse and Detection of M365D Live Response for privilege escalation on Control Plane (Tier0) assets](https://www.cloud-architekt.net/abuse-detection-live-response-tier0/) outlines leveraging Defender's live response feature to execute code on endpoints.

The way Microsoft has laid out Microsoft Defender for Endpoint and Defender 365 is a little confusing. Live response can be accessed via https://api.securitycenter.micorsoft.com scope with the appropriate permissions.
Defender 365 also outfits blue teams with threat hunting capabilities. This comes in the form of advanced threat hunting in the defender 365 portal, which provides tables to query from to peform threat hunting operations.
This includes exchange emails, defender for endpoint alerts and incidents and even device network, file and process events. KQL is the query language used to conjure up results. Advanced threat hunting queries can also be
ran through the [graph api](https://learn.microsoft.com/en-us/graph/api/security-security-runhuntingquery?view=graph-rest-1.0&tabs=http). However, this is access through the https://graph.micrsoft.com scope. 

If you manage to grab an access token with the ThreatHunting.Read.All API permission or the credentials of an entra user assigned with the security reader, security operator or security administrator entra role. Then you're presented the possibility of enumerating resources within the tenant via advanced threat hunting. The [exposure graph](https://learn.microsoft.com/en-us/security-exposure-management/query-enterprise-exposure-graph) within advanced hunting can help with this. Specifically [ExposureGraphNodes](https://learn.microsoft.com/en-us/defender-xdr/advanced-hunting-exposuregraphnodes-table) and [ExposureGraphEdges](https://learn.microsoft.com/en-us/defender-xdr/advanced-hunting-exposuregraphedges-table). 

*Note, that this works even if the principal doesn't have any subscriptions or resources assigned to it.*

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

This is great and all, but there's still not any convienient way to interact with defender/security center programatically. Analyzing requests in burp suite I discovered that everytime a query was executed the request would be sent to the following URL, https://security.microsoft.com/apiproxy/mtp/huntingService/queryExecutor?useFanOut=false. In the [following article from Falcon Force](https://medium.com/falconforce/microsoft-defender-for-endpoint-internals-0x04-timeline-3f01282839e4), this is described as an apiproxy that Microsoft has put in place to prevent interaction in this manner. It is also mentioned that the folks at Falcon Force were able to write a Python script that works around the apiproxy. However, this isn't shared, linked or present anywhere in their github repo.

So, back to playing around with trusty old Burp suite. Upon replaying the request multiple times and removing things from it. I discovered that the bare minimum you need to make the request work is the following:

### Request Headers
* Cookie: sccauth=...
* X-Xsrf-Token
* Content-Type: application/json

### Post Request Parameters
* QueryText="KQL query"
* StartTime="YYYY-mm-ddTHH:MM:sssZ"
* EndTime="YYYY-mm-ddTHH:MM:sssZ"
* MaxRecordCount=10

In theory you could login to the security and compliance center portal via browser and pull the needed header values with inspect element and then send the post request with the desired values for the parameters. However, to me, this is a major inconvience. What else can we try? Well, I tried mimicking the interactive login flow with python request sessions to no avail. Feel free to give this a shot yourself and please let me know if you get this working.
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

# Despite everything looking like it is setup correctly the response is a 440 - signin reason timeout
```

Okay, that didn't work. The last resort is using selenium to automate the interactive login flow to security.microsoft.com. The ability to run threat hunting queries in with Habu2 has been added using this method. I'm currently looking into the ability to interact with the live response feature via the apiproxy.
