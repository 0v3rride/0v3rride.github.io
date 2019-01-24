# How Fat-Fingering a URL Led Me To Discovering An Undisclosed Vulnerability
![Linkedin]() ![Github]()

### Preface

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
   
### Potential Impact & Attack Vectors
