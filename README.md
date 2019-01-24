# How Fat-Fingering a URL Led Me To Discovering An Undisclosed Vulnerability
![Linkedin](https://cdn1.iconfinder.com/data/icons/logotypes/32/square-linkedin-512.png) ![Github](https://cdn1.iconfinder.com/data/icons/logotypes/32/github-512.png)

### Preface

```markdown
 https://host.example.com/runJob.html?__76&jobId=**#**__
```
However, I discovered that simply removing that small portion (&76) from the URL had no affect on the content in the webpage that were produced from a response.

```markdown
 https://host.example.com/runJob.html?jobId=__#__
```
