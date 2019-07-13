![Linkedin](Post%20Images/linkedin.png) [Linkedin](https://www.linkedin.com/in/ryangore/) | ![Github](Post%20Images/github.png) [Github](https://github.com/0v3rride)

# Enum4LinuxPy
_____________________________________________________________________

![e4lpy](Post%20Images/e4lpy-imgs/e4lpy.PNG)

Enum4Linux is one of my favorite tools for enumerating a remote host via SMB and RPC. Some other great tools for SMB enumeration are:

* nmblookup
* nbtscan
* NSE scripts
* Metasploit auxiliary modules
* SMBMap
* Smbclient
* Rpcclient
* net
* etc.

I really like enum4linux, because it's simple to use and really encompasses most of those tools listed above into one solid package. However, I noticed in the last couple of months when using enum4linux that it was returning garbled or mangled output. It's quite difficult to parse for useful information in a sea of errors and warnings from an interpreter. One of the errors that I've seen in various forums that people complain about is the `Use of uninitialized value $os_info in concatenation (.) or string at /bin/enum4linux line 464.` warning/error. In certain situations the output will not return the samba or OS version info due to this error. You can read more about this issue [here](https://github.com/portcullislabs/enum4linux/issues/5). It's even referenced in an entry in Kali's [bug tracker](https://bugs.kali.org/view.php?id=4495). Unfortuantely, I've found the solution that g0tmi1k has shared doesn't help me in my situation. This issue is only one of [7 total](https://github.com/portcullislabs/enum4linux/issues) outstanding issues, most of which have been open for over a year. I'm not sure if this means that Mark Lowe or Port Cullis Labs are not maintaing it anymore.

So I decided to port the code over to the Python language. Enum4Linux is a simple program that is a wrapper around the following tools that are installed on a typical Kali instance:

* nmblookup
* net
* rpcclient
* smbclient
* polenum
* ldapsearch

In layman's terms, the script makes command line calls ([safely](https://security.openstack.org/guidelines/dg_use-subprocess-securely.html), so the chance of command shell injection happening is slim to none) to the tools listed above, gathers the output, formats it all nice and pretty and then spits it out to your terminal for you to read.

You can clone it from https://github.com/0v3rride/Enum4LinuxPy. I've used Enum4LinuxPy in couple of scenarios for the past week and it seems to be working the way it was intended. When time permits, I would like to extend it's functionality, hopefully with different rpcclient commands, net commands or Python rpc and smb apis. If you encounter any problems when using Enum4LinuxPy, then you can open an issue on the project's github page. I will attend to it as soon as possible when time permits. Also feel free to submit a new pull request if you feel something should be changed, added or fixed.
