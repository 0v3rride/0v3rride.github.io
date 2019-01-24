![Linkedin](Site%20Pictures/linkedin.png)[Linkedin](https://www.linkedin.com/in/ryangore/)
![Github](Site%20Pictures/github.png)[Github](https://github.com/0v3rride)

# How Fat-Fingering a URL Led Me To Discovering An Undisclosed Vulnerability (CVE 2019-6716)


### Preface

![LogonBox Limited Access Manager](Site%20Pictures/nervepoint_vx.png)

############### [Image provided by LogonBox Limited & Hypersocket Software](https://www.hypersocket.com/en/products/password-self-service)

```markdown
 https://host.example.com/runJob.html?76&jobId=#
```
However, I discovered that simply removing that small portion (&76) from the URL had no affect on the content in the webpage that was produced from a response. This means there was no need for finding a pattern or predicting the number that would be produced every time a request was sent. Do you know what that means?

```markdown
 https://host.example.com/runJob.html?jobId=#
```
This means that an attacker can easily automate requests with a tool such as Burpsuite's intruder tool. This made it easier to identify what types of jobs were running on the back-end of the server that an unauthenticated attacker could access from the main public page of the web application.

While automating the requests with the Burpsuite intruder tool, I discoved a multitude of information that shouldn't be accessible to an unauthenticated user. This information included:
 * The username schema that the orginization uses for internal Active Directory usernames
 * Enumeration of internal Active Directory usernames
 * Enumeration of internal Active Directory group names (in this specific instance)
 * AD users who have recently changed their passwords
 * Back-end server jobs running with the possibility to alter their states/status
   * Sychronization jobs
   * Backup jobs
   
### Potential Opportunites To Leverage
##### Password Spraying/Reverse Brute forcing
The username enumeration aspect of this issue makes this activity much more trivial since we have half of the equation; all we need is a valid password. Believe or not people still use awful passwords. To make this aspect even worse, some people like to reuse their password(s) across multiple online services and places whether it's at work, school etc. These things are what makes password spraying the best thing since sliced bread. Keep in mind that one would have to be cognizant of the account lockout policies in place in the internal AD enviroment since Access Manager ties in with it. 
 
 **If you have the ability to turn on any multi-factor authentication for anything I would strongly suggest you do so immediately!!!**
 
##### Account Take Over & Gaining A Foothold In The On Prem AD Enviroment or Cloud Services 
Access Manager, in my opinion, also doesn't have a great way of letting users reset their password if they forgot it or if they get locked out of their account. I'm not a hundered precent sure if it was this specific instance or for all Access Manager instances. Let me explain, for a user to reset their password or unlock their account they simply need to answer 3 of 5 secret personal questions. The same problem concerning the way users choose their passwords also applies to how users pick their super-duper secret answers to their secret questions.

![Secret Questions](Site%20Pictures/password-reset.jpg)

 * **Secret question:** What was the first car you owned?
 * **Super-duper secret answer:** ford, toyota, rav4, corolla, etc.
 * **A potentially better answer:** sterling silver general motors company sierra, etc. _(Plus one if answer is case sensitive)_
 * **An even better answer:** boeing F/A-18E/F Super Hornet _(That's not a car, that's a twin-engine, carrier capable, multirole fighter aircraft)_
 
Do you see what I mean? A wordlist of car makes and models could easily be generated. To top it all off, Access Manager allows users to set their own secret questions (what is your favorite color?). Not to mention at this point an attacker could make reasonable inferences or peform OSINT on individuals with the list of usernames they enumerate. An attacker can essentially make the process of resetting or unlocking an Active Directory account through a web portal trivial by using simple guessing, inferences, OSINT techniques and tools.

##### Possible Denial of Service
During the automation of the requests, I also discovered that possibly had the opportunity to cancel backup jobs, synchronizaiton jobs and other jobs running on the back-end. This could possibly affect an orginization in multiple ways. Such as users not having access to their accounts, etc.
