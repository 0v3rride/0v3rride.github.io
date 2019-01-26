![Linkedin](Site%20Pictures/linkedin.png)[Linkedin](https://www.linkedin.com/in/ryangore/)
![Github](Site%20Pictures/github.png)[Github](https://github.com/0v3rride)

# How Fat-Fingering A URL Led Me To Obtaining My First CVE ID (2019-6716)

### Preface

LogonBox Limited's (formerly Nervepoint technologies) and Hypersocket software's Access Manager is a virtual web application appliance that intergrates with an on premise [Active Directory environment](https://docs.logonbox.com/app/manpage/en/article/413106/Connecting-to-an-Active-Directory) and off-premise solutions such as Google's G Suite service. In turn, this allows users part of an orginization that utilizes this product to manage their Active Directory and cloud accounts all through Access Manager. This allows orginizations that may not have enough resources to allocate to a help desk group cut back on help calls that involve the mundane and sometimes time consuming process of resetting a user's password or unlocking an account. However, I've noted some concerns below in how some these mechanisms for resetting and unlocking Active Directory accounts via a publicly accessible web application works.

##### Official Summary Of Issue
> An unauthenticated Insecure Direct Object Reference (IDOR) vulnerability in LogonBox Limited's (formerly Nervepoint Technologies) Access Manager web application allows a remote attacker to enumerate internal Active Directory usernames. It also allows for the possibility to enumerate Active Directory group names and altering of back-end server jobs (backup and synchronization jobs) depending on the configuration of the system. This is done via the manipulation of the jobId HTTP parameter in an HTTP GET request. This issue affects Access Manager versions >= 1.2 <= 1.4-RG3 and has been rectified in versions >= 1.4-RG4.

![LogonBox Limited Access Manager](Site%20Pictures/nervepoint_vx.png)

###### [The above image belongs to LogonBox Limited & Hypersocket Software](https://www.hypersocket.com/en/products/password-self-service)

Finding the issue in the web application was really quite simple after accidentally mistyping the URL. It took me a couple of seconds to realize what was happening as I wasn't purposefully auditing the web application for issues. As I incremented the number in the jobId parameter, Active Directory names and potentially AD group names were being returned in the response. I could see users who had recently changed their passwords and then the synchronization jobs running for different AD groups that these users were part of. I was also presented with the chance to cancel these sync jobs, backup jobs and other types of jobs if I wanted **(Spoiler alert: I didn't try to see if I could).** I informed the Hypersocket software team of this issue immediately and I received a prompt response that they were able to rectify the issue and push out the patch to their customer base.

I was informed after further testing by the Hypersocket team that they were only able to enumerate AD usernames (2 to 3 out of a sample of 150). They weren't able to enumerate AD group names or cancel any jobs running on the back-end. However, in my case, I was able to enumerate many more AD usernames. This is most likely due to the fact that the sample size I was working with was much larger. I suspect that the possibility to enumerate AD group names and cancel jobs running on the back-end may be dependent on the configuration of a specific instance of Access Manager that is running.

### Proof Of Concept
Again, this issue was trivial to identify. It has the classic makings of an Insecure Direct Object Reference. Simply changing the integer value for the jobId HTTP GET parameter to view information that an unauthenticated user probably shouldn't be able to view did the trick.

```markdown
 https://host.example.com/runJob.html?76&jobId=#
```
Furthermore, I discovered that simply removing that small portion (&76) from the URL had no effect on the content in the webpage that was produced from a response. This means there was no need for finding a pattern or predicting the integer value that would be produced every time a request was sent. Do you know what that means?

```markdown
 https://host.example.com/runJob.html?jobId=#
```
This means that an attacker can easily automate requests with a tool such as PortSwigger's Brupsuite intruder module. This made it easier to identify what types of jobs were running on the back-end of the server that an unauthenticated attacker could access from the main public page of the web application.

While automating the requests with the Burpsuite intruder tool, I discovered a multitude of information that shouldn't be accessible to an unauthenticated user. This information included:
 * The username schema that the organization uses for internal Active Directory usernames
 * Enumeration of internal Active Directory usernames
 * Enumeration of internal Active Directory group names (in this specific instance)
 * Enumeration of AD users who have recently changed their passwords
 * Enumeration of back-end server jobs running with the possibility to alter or interact with their states/status
   * Synchronization jobs
   * Backup jobs
   * Rebuild jobs
   
   
### Potential Opportunites To Leverage From The Perspective Of An Attacker
##### Password Spraying/Reverse Brute forcing
The username enumeration aspect of this issue makes this activity much more trivial since we have half of the equation; all we need is a valid password. Believe or not people still use awful passwords. To make this aspect even worse, some people like to reuse their password(s) across multiple online services, work account, school account, etc. These things are what makes password spraying the best thing since sliced bread. Keep in mind that one would have to be cognizant of the account lockout policies in place in the internal or Azure AD environments since Access Manager can tie in with either of those. 
 
 **If you have the ability to turn on any multi-factor authentication for anything I would strongly suggest you do so immediately!!!**
 
##### Account Take Over & Gaining A Foothold In The On Prem AD Enviroment or Cloud Services 
In my case, the running instance of Access Manager also didn't have a great way of letting users reset their password if they forgot it or if they get locked out of their account. For a user to reset their password or unlock their account they simply need to answer 3 of 5 secret personal questions. The same problem concerning the way users choose their passwords also applies to how users pick their super-duper secret answers to their secret questions.

![Secret Questions](Site%20Pictures/password-reset.jpg)
###### Generic screenshot pulled from a Google search

 * **Secret question:** What was the first car you owned?
 * **Super-duper secret answer:** ford, toyota, rav4, corolla, etc.
 * **A potentially better answer:** sterling silver general motors company sierra, etc. _(Plus one if answer is case sensitive)_
 * **An even better answer:** boeing F/A-18E/F Super Hornet _(That's not a car, that's a twin-engine, carrier-capable, multirole fighter aircraft)_
 
Do you see what I mean? A wordlist of car makes and models could easily be crafted. Access Manager also allows users to set their own secret questions. _What is your favorite color?_ Not to mention at this point an attacker could make reasonable inferences or perform OSINT on individuals with the list of usernames they enumerate. The process of resetting or unlocking an Active Directory account through a web portal can be made trivial by using simple guessing, inferences, OSINT techniques and tools. 

In addition, Access Manager allows an administrator to enable [multiple methods for authentication](Allowing multiple Authentication Methods to be active for login processes in Access Manager) such as OTP, PIN, SMS, etc. At least one to two alternative [authentication methods](https://docs.logonbox.com/app/manpage/en/article/532236/Authentication-Basics:-Configuring-and-managing) should be used to replace the secret questions method in this case.

**It should be noted that using secret questions and answers _NOT_ a valid form of MFA nor is it a good way to verify the identity of a user! The password and answers to the secret questions are both something the user knows."

##### Possible Denial of Service
During the automation of the requests, I also discovered that possibly had the opportunity to cancel backup jobs, synchronization jobs and other jobs running on the back-end. This could possibly affect an organization in multiple ways. Such as users not having access to their accounts, etc.


### Conclusion
I'd also like to mention that LogonBox Limited & Hypersocket Software allow for the integration of the [yubico key](https://www.yubico.com/works-with-yubikey/catalog/logonbox/) for true multi-factor authentication. I'm not entirely sure if this can integrate with the on prem Access Manager product as opposed to the cloud solution.


##### Credits
 * [A very helpful guide from Michael Benich that summarizes the CVE process for beginners](https://warroom.rsmus.com/beginners-guide-cve-process/)
 * [Official CVE MITRE ID request form](https://cveform.mitre.org/)
 * [**A special thanks to the team at LogonBox Limted and Hypersocket software for effective communication and cooperation**](https://docs.hypersocket.com/app/manpage/en/article/539661/Unauthenticated-Insecure-Direct-Object-Reference-(IDOR)-Found-in-Access-Manager)
 * [CVE 2019-6716](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-6716)
