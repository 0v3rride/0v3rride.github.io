![Linkedin](Site%20Pictures/linkedin.png)Linkedin 
![Github](Site%20Pictures/github.png)Github

# How Fat-Fingering a URL Led Me To Discovering An Undisclosed Vulnerability


### Preface
![LogonBox Limited Access Manager]()

```markdown
 https://host.example.com/runJob.html?__76&jobId=**#**__
```
However, I discovered that simply removing that small portion (&76) from the URL had no affect on the content in the webpage that was produced from a response. This means there was no need for finding a pattern or predicting the number that would be produced every time a request was sent. Do you know what that means?

```markdown
 https://host.example.com/runJob.html?jobId=__#__
```
This means that one can easily automate requests with a tool such as Burpsuite's intruder tool. This made it easier to identify what types of jobs were running on the back-end of the server that an unauthenticated attacker could access from the main public web page. 

While automating the requests with the Burpsuite intruder tool, I discoved a multitude of information that shouldn't be accessible to an unauthenticated user. This information included:
 * The username schema that the orginization uses for internal Active Directory usernames
 * Enumeration of internal Active Directory usernames
 * Enumeration of internal Active Directory group names (in this specific instance)
 * AD users who have recently changed their passwords
 * Back-end server jobs running with the possibility to alter their states/status
   * Sychronization jobs
   * Backup jobs
   
### Potential Opportunites To Leverage
 1. **Password Spraying/Reverse Brute forcing** - The username enumeration aspect of this issue makes this activity much more trivial since we have half of the equation; all we need is a valid password. Believe or not people still use awful passwords. To make this aspect even worse, some people like to reuse their password(s) across multiple online services and places whether it's at work, school etc. These things are what makes password spraying the best thing since sliced bread. Keep in mind that one would have to be cognizant of the account lockout policies in place in the internal AD enviroment since Access Manager ties in with it. 
 
 **If you have the ability to turn on any multi-factor authentication for anything I would strongly suggest you do so immediately!**
 
 2. **Account Take Over & Gaining A Foothold In The On Premises AD Enviroment** - Access Manager, in my opinion, also doesn't have a great way of letting users reset their password if they forgot it or if they get locked out of their account. I'm not a hundered precent sure if it was this specific instance or for all Access Manager instances. 
 
Let me explain, for a user to reset their password or unlock their account they simply need to answer 3 of 5 secret personal questions. The same problem concerning the way users choose their passwords also applies to how users pick their super-duper secret answers to their secret questions.

![Secret Questions](Site%20Pictures/password-reset.jpg)

 * **Secret question:** What was the first car you owned?
 * **Super-duper secret answer:** ford, toyota, rav4, corolla, etc.
 * **A potentially better answer:** sterling silver general motors company sierra, etc. (plus one if answer is case sensitive)
 
Do you see what I mean? A wordlist of car makes and models could easily be generated. To top it all off, Access Manager allows users to set their own secret questions (what is your favorite color?). Not to mention at this point an attacker could make reasonable inferences or peform OSINT on individuals with the list of usernames they enumerated Eventual this could end up leading to the possibility of discovering that user drives a certain make, model, color and year of car. An attacker can essentially make the process of resetting or unlocking an Active Directory account through a web portal trivial by using simple guessing, inferences, OSINT techniques and tools.

 3. **Possible Denial of Service** - During the automation of the requests, I also discovered that possibly had the opportunity to cancel backup jobs, synchronizaiton jobs and other jobs running on the back-end. This could possibly affect an orginization in multiple ways. Such as users not having access to their accounts, etc.
