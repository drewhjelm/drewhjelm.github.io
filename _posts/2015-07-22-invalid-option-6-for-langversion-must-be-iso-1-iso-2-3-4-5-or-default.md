---
layout: post
title: Invalid option ‘6’ for /langversion; must be ISO-1, ISO-2, 3, 4, 5 or Default
---

I started working with Visual Studio 2015 tonight and ran into an error. The website I’m building is a test site for Azure, but the target framework for the site was .NET Framework 4.6 (which Azure does not yet support for Web Apps). I downgraded the target framework to 4.5.2 and the website compiles correctly, but when I go to run in browser, I receive an error:

{% highlight powershell %}
Invalid option '6' for /langversion; must be ISO-1, ISO-2, 3, 4, 5 or Default
Source Error:
[No relevant source lines]
Source File: Line: 0
{% endhighlight %}

Initially I tried changing the version in Properties > Build > Advanced, but that was not solving the issue.

This error is corrected by changing compiler settings in the web.config.

{% highlight powershell %}
<compiler language="c#;cs;csharp" extension=".cs" type="Microsoft.CSharp.CSharpCodeProvider, System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089" warningLevel="4" compilerOptions="/langversion:6 /nowarn:1659;1699;1701">
{% endhighlight %}

Changing langversion to 5 will allow the site to compile on Azure and in .NET 4.5.2.