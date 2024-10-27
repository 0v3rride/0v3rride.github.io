# BYOI with JavaScript

![Github](Post%20Images/github.png) [Github](https://github.com/0v3rride)

## Background

Bring Your Own Interpreter is technique thas has been employeed by red teams to achieve code execution. The concept of embedding your favorite scripting languages interpreter into a payload becomes more attractive. Especially, as more runtime, logging and insight defenses are built into PowerShell due to its widespread abilities. The SilentTrinity framework already makes use of the concept by utilizing the [Boo scripting language](https://github.com/byt3bl33d3r/SILENTTRINITY/tree/master/silenttrinity/core/teamserver/data). One could also use [IronPython, IronRuby, F#,](https://www.irongeek.com/i.php?page=videos/derbycon9/1-17-red-team-level-over-9000-fusing-the-powah-of-net-with-a-scripting-language-of-your-choosing-introducing-byoi-bring-your-own-interpreter-payloads-marcello-salvati) and so on to achieve this.

## Discovery (or so I thought)

I stumbled across an interesting project that is utitilized by NoSQL databases software called RavenDB. RavenDB exposes an administration console (and fields for UNC path injection to steal Net-NTLM hashes) that [_"allows you to execute arbitrary JavaScript code"_](https://ravendb.net/docs/article-page/3.5/Csharp/studio/management/administrator-js-console). Interesting, how is an application such as RavenDB able to execute JavaScript code? I found out the answer to that pretty quickly by consulting the [documentation](https://ravendb.net/docs/article-page/4.2/csharp/server/kb/javascript-engine). RavenDB uses [Jint](https://github.com/sebastienros/jint), an ECMAScript 5.1 compliant JS interpreter that can be embedded in .NET applications. The DLL assembly can be downloaded from the NuGet package manager via Visual Studio and then simply implemented into a .NET project. I was only made aware a couple of days after my discovery that Marcello (@byt3bl33d3r) had already noted that Jint could also be used in his talk [_"IronPython... OMFG Introducing BYOI Payloads (Bring Your Own Interpreter)"_](https://hackinparis.com/data/slides/2019/talks/HIP2019-Marcello_Salvati-Ironpython_Omfg.pdf).

## Turning a Spoon Into a Weapon

Jint is relatively simple to use. [Use .NET to execute arbitrary JavaScript code. Or use Jint to invoke .NET via JavaScript](https://blog.codeinside.eu/2019/06/30/jint-invoke-javascript-from-dotnet/). In the below example, I create a JavaScript function named JExec and test to reference the System.Diagnostics.Process.Start method and a System.Diagnostics.Process object respectively.

```
  //Engine e = new Engine();
  
  //e.SetValue("JExec", new Func<string, string, System.Diagnostics.Process>(System.Diagnostics.Process.Start));
  //e.Execute("JExec('powershell.exe', '-exec bypass -noexit -c whoami');");

  //can represent a new instance of a .net object or method via a delegate type like action<>, action, or func<>
  
  //e.SetValue("test", new System.Diagnostics.Process());
  //e.Execute("test.Start('calc.exe')");
```

Obviously, you can get a lot more crafty with it, but I kept it simple for demonstrative purposes. Many know that JavaScript doesn't really expose any socket APIs. So, trying to craft a bind or reverse shell payload in JS is kind of off the table. I'm not aware of any JavaScript library that exists for socket programming. However, if you find one, then you can read the source code from the JS file and execute it using an instance of the Jint.Engine class.
