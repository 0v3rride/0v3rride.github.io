![Linkedin](Post%20Images/linkedin.png) [Linkedin](https://www.linkedin.com/in/ryangore/) | ![Github](Post%20Images/github.png) [Github](https://github.com/0v3rride)

# Pwning A Pentesting Tool
_____________________________________________________________________

For some reason looking at other people's source code makes me get all warm and fuzzy inside. Kinda of like the description a Millenial hipster gives you. You know, when they tell you how they sat down and read a 'good book' with a cup of gluten-free, 100% non-GMO, Rainforest Alliance Certified Coffee. 

Reading source code from other individuals or groups can give you an insight into a lot of things like:
  * New syntax and syntactical methods that you never knew existed in a language
  * How to get code to execute by bypassing AV, countermeasures, etc.
  * What not to do when developing an application of any sort
  * What the drug or alcohol choice of the individual/group was during development ;)
  
## Some Of Python's Dangerous Functions & Methods
In my opinion, Python is a fantastic language. It's one of my favorites along side with PowerShell, and C#. However, like all computer languages it doesn't come without it's flaws or dangerous functions and methods. You can read more about some of those dangerous functions and methods in the below links:
 1. https://www.kevinlondon.com/2015/07/26/dangerous-python-functions.html
 2. https://www.kevinlondon.com/2015/08/15/dangerous-python-functions-pt2.html
 3. https://security.openstack.org/guidelines/dg_use-subprocess-securely.html
 4. https://security.openstack.org/guidelines/dg_avoid-shell-true.html
 5. https://medium.com/python-pandemonium/a-trap-of-shell-true-in-the-subprocess-module-6db7fc66cdfd
 6. https://docs.python.org/3.7/library/subprocess.html#popen-objects
 
## The Vulnerability
The vulnerability itself is not super-duper serious as a vast majority of Linux machines will probably not have this tool installed. While working on a CTF problem involving SIDs and looking for another tool to compare with Impacket's lookupsid.py script, I came across a tool written by Dave Kennedy (ReL1K) called RidEnum.py. I was interested in how both tools were obtaining the Domain SID, but that's not really important. However, I noticed in the source code on the [github page](https://github.com/trustedsec/ridenum/blob/master/ridenum.py) that the tool was using a Subprocess.Popen object to execute a shell command. This appears on lines 62 and 78. 

There are two arguments that helped me to perform command injection. If you look closely, you'll note that the variable `command` which will store the string representation of the command to be executed takes the `auth` (or optional username argument) and the IP argument and then places them into the string using the "old style" string formatting operator (`%`).

Two things can be taken in to consideration at this point (links 3, 4 and 5 above do a great job explaining the problems):

1. The `shell` parameter in both calls are set to `true`. That's a no no, especially if you allow a user to supply input to your script or program. In simple terms, this tells Python to execute the string containing the command(s) as if you were doing it in a bash shell. Lets look at an example:

Here's the help documentation for ridenum.
```
Example: ./ridenum.py 192.168.1.50 500 50000 /root/dict.txt /root/user.txt

Usage: ./ridenum.py <server_ip> <start_rid> <end_rid> <optional_username> <optional_password> <optional_password_file> <optional_username_filename>
```
Let's execute ridenum and inject the bash command `id`.

```
./ridenum.py 10.10.10.10;id; 100 1000 0v3rride
```

Result (I'm executing this on my Kali machine locally, that's why I'm root):
```
uid=0(root) gid=0(root) groups=0(root)
bash: 100: command not found
```
Note the error below the output of the `id` command and how I placed the `id` command with the `IP` argument. Why do I need two `;`? If this were executed as `;id 100 1000 0v3rride` then everything after `id<space>` is treated as an argument to the `id` command. So in this case an error would be `id: extra operand ‘1000’`. One can run the bash command `id` and supply it a username as an argument (`id <username>`). 

Let's get the id information for the account `ntp`.
```
./ridenum.py 10.10.10.10;id ntp;100 1000 0v3rride

uid=111(ntp) gid=114(ntp) groups=114(ntp)
bash: 100: command not found
```
 
