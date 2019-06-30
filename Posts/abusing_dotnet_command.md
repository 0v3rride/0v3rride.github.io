# Abusing The dotnet Command

![Linkedin](Post%20Images/linkedin.png) [Linkedin](https://www.linkedin.com/in/ryangore/) | ![Github](Post%20Images/github.png) [Github](https://github.com/0v3rride)

## Background

`dotnet.exe` is an executable that can be ran from a PowerShell or command prompt to manage .NET source code or projects. This includes building a project, adding a reference/assembly to a .NET project and more. Just issue the following command to see for yourself `dotnet /?`. However, it also has a great additional feature that allows one to execute a .NET Core application created in Visual Studio. This means you can shoot off a .dll file that is produced as the result of building an application in VS. I've only tested this with the C# payload (.NET Core console app) I created in my instance of VS. I'm not sure if you can just run any .dll file using the dotnet command.

## Caveats

* The targeted remote host needs to have some version of Microsoft Visual Stuido installed.

* The target remote host needs to have the .NET Framework Runtime version 3.5 installed.

## How to use it

To execute a .dll payload it's pretty straight forward - `dotnet C:\path\to\dll\fileorpayload.dll`. One could probably get a lot more crafty with the additional [options](https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet?tabs=netcore21) that the `dotnet` command exposes.

![dll-exec-dotnet](Post%20Images/Abusing-dotnet-imgs/dotnet-exec.png)
