# Offender 365

![Linkedin](Post%20Images/linkedin.png) [Linkedin](https://www.linkedin.com/in/ryangore/) | ![Github](Post%20Images/github.png) [Github](https://github.com/0v3rride)

## Background
I recently completed the MCRTP exam from Pwnedlabs. Finally an offensive Azure course on the same tier as those offered by Offensive Security. The course itself included a lot of information that I feel like the SANS GCPN glossed over.
During the course there were a couple of areas in Azure/O365 that piqued my intrests and wanted to circle back on after completing the exam.

* What makes continous access elevation tokens special?
* Is there a way to acquire a token via the OAuth flow API with all the necessary scopes to access storage accounts, key vaults with one access token to rule them all?
  * The token should have similar permissions or scopes to the one you get via interactive login using Connect-AzAccount.
  * Is this where FOCI comes into play? What exactly is FOCI?
    
I'm still digging for answers in the points above, but there's another questions I've been able to find the answer to myself. Are there any other services or scopes that can be used as attack paths in an orginization's tenant?
The simple answer is yes, besides the Graph . There's two main services/resources that I've been looking into. 

I remember Ian, our gracious course instructor and subject expert for the MCRTP, mentioning during one of the sessions that he was 
looking into Intune as an attack path. Having worked with Intune a little. I know with the appropriate permissions you can execute code remotely (Powershell) on a managed device of your choice. Which in theory should allow you
laterally move from the orginazation's cloud enviroment to on-prem and possibly vice versa.

### Defender 365
That's great and all, but I'm feeling a little more devious. What if we can use Defender 365 in ways it wasn't supposed to be used. Well, someone's already beat me to the punch in one interesting aspect. Research by Thomas Naunheim titled, [Abuse and Detection of M365D Live Response for privilege escalation on Control Plane (Tier0) assets](https://www.cloud-architekt.net/abuse-detection-live-response-tier0/) outlines leveraging Defender's live response feature to execute code on endpoints.

Defender 365 also outfits blue teams with threat hunting capabilities. This comes in the form of advanced threat hunting in the defender 365 portal, which provides tables to query from to peform threat hunting operations.
This includes exchange emails, defender for endpoint alerts and incidents and even device network, file and process events. KQL is the query language used to conjure up results. Advanced threat hunting queries can also be
ran through the [graph api](https://learn.microsoft.com/en-us/graph/api/security-security-runhuntingquery?view=graph-rest-1.0&tabs=http). If you manage to grab an access token with the ThreatHunting.Read.All permissions, then you're presented the possibility of enumerating resources within the tenant via advanced threat hunting. The [exposure graph](https://learn.microsoft.com/en-us/security-exposure-management/query-enterprise-exposure-graph) within advanced hunting can help with this. [ExposureGraphNodes](https://learn.microsoft.com/en-us/defender-xdr/advanced-hunting-exposuregraphnodes-table) and [ExposureGraphEdges](https://learn.microsoft.com/en-us/defender-xdr/advanced-hunting-exposuregraphedges-table). Note, that this works even if the principal doesn't have any subscriptions or resources assigned to it.

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
