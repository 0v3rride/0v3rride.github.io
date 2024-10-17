# Offender 365

![Linkedin](Post%20Images/linkedin.png) [Linkedin](https://www.linkedin.com/in/ryangore/) | ![Github](Post%20Images/github.png) [Github](https://github.com/0v3rride)

## Background
I recently completed the MCRTP exam from Pwnedlabs. Finally an offensive Azure course on the same tier as those offered by Offensive Security. The course itself included a lot of information that I feel like the SANS GCPN glossed over.
During the course there were a couple of areas in Azure/O365 that piqued my intrests and wanted to circle back on after completing the exam.

* What makes continous access elevation tokens special?
* Is there a way to acquire a token via the OAuth flow API with all the necessary scopes to access storage accounts, key vaults with one access token to rule them all?
  * The token should have similar permissions or scopes to the one you get via interactive login using Connect-AzAccount.
  * Is this where FOCI comes into play? What exactly is FOCI?
    
I'm still digging for answers in the points above, but there's another questions I've been able to find the answer to myself. Are there any other services or scopes that can be used as attack paths in an orginizations tenant?
The simple answer is yes, besides the Graph . There's two main services/resources that I've been looking into. I remember Ian (our gracious course instructor and subject expert for the MCRTP) mentioning during one of the sessions that he was 
looking into Intune as an attack path. Having worked with Intune a little. I know with the appropriate permissions you can execute code remotely (Powershell) on a managed device of your choice, which in theory should allow you
laterally move from the orginazations cloud enviroment to on-prem.

## Discovery (or so I thought)
