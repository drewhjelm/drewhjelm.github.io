---
layout: post
title: Easy IIS SSL Hardening
---

Recently I found a need for a one-size-fits-all script to harden a web server. I wanted to be able to take a fresh server and get the server to a high rating on [SSL Labs testing](https://www.ssllabs.com/ssltest/) as easily as possible. The challenge with IIS setup (as opposed to Apache or Nginx) is the relative lack of scripts already published for improving security.

In the past, I’ve used a utility called [IIS Crypto](https://www.nartac.com/Products/IISCrypto/) to harden IIS security, but running a GUI application on every web server as opposed to running a script is less efficient. I don’t have any reason to distrust the makers of the IIS Crypto application, but it is also nice to have complete visibility into the script I’m running to secure a server.

Alternatively, you can hand edit the XML files controlling your web server, but using this script will save time over editing those files.

What I would like to do is the following:

* Improve SSL by disabling SSLv3 and increasing the cipher strength of the server and implement Perfect Forward Secrecy
* Make the website HTTPS only
* Implement HTTP Strict Transport Security (HSTS)
* Remove Server Identification
* Prevent Clickjacking

Prerequisites for this method of server hardening:

* Windows Server 2008, 2008 R2, 2012, or 2012 R2 with Internet Information Systems (IIS) installed on the server
* Your SSL Certificate installed and bound to a website on the server
* [URL Rewrite 2.0](http://www.iis.net/downloads/microsoft/url-rewrite) – This module allows you to make changes to how IIS handles different requests to the web server. In particular, we want to change requests from HTTP to HTTPS and rewrite certain server variables.

**tl;dr: the entire script is on [GitHub](https://github.com/drewhjelm/iis-hardening).**

## Improve SSL and Add Perfect Forward Secrecy

Some of the most recent SSL vulnerabilities have been due to older versions of the SSL protocol, SSLv3, which was built back in the 1990s. The exploits have allowed attackers to downgrade a server’s encryption from the relatively weak SSLv3 to no security at all. The best way to mitigate this is to disable SSL and rely on Transport Layer Security (TLS), the updated version of SSL, to secure traffic.

In addition to disabling SSLv3, it is necessary to disable weak ciphers and key exchange mechanisms to improve security. Weaker ciphers, such as RC4, can [more easily be exploited](http://www.securityweek.com/new-attack-rc4-based-ssltls-leverages-13-year-old-vulnerability) than newer ciphers, so it is best to disable as many old ciphers as can be withstood by your end users. Older operating systems your users may still have, like Windows XP, may not be able to handle newer ciphers using Elastic Curve Diffie-Hellmann Exchange (ECDHE) and Advanced Encryption Standard (AES).

This standard should attempt to negotiate 256-bit encryption with as many clients as possible.

Another improvement to web server security is to ensure that Perfect Forward Secrecy (PFS) is enabled. PFS ensures that a web server’s keys used to secure traffic with clients [cannot be decrypted by outsiders](https://www.eff.org/deeplinks/2013/08/pushing-perfect-forward-secrecy-important-web-privacy-protection).

I will be incorporating [Alexander Hass’ Powershell script](https://www.hass.de/content/setup-your-iis-ssl-perfect-forward-secrecy-and-tls-12) to improve SSL for this project.

## Make the server HTTPS only

Making traffic to a web server only travel over HTTPS ensures that all traffic is encrypted over the wire.

This is probably only necessary for sites handling sensitive information, but at the very least any login (username + password) should be encrypted.

This should be configured with a Rewrite rule using URL Rewrite. The rule itself can be found on [OWASP](https://www.owasp.org/index.php/HTTP_Strict_Transport_Security#IIS), but this still requires either addition through GUI or in the web.config file. Here’s the script to create this rewrite rule.

{% highlight powershell %}
appcmd set config -section:system.webServer/rewrite/rules /+"[name='HTTPS_301_Redirect',stopProcessing='False']" /commit:apphost
appcmd set config -section:system.webServer/rewrite/rules "/[name='HTTPS_301_Redirect',stopProcessing='False'].match.url:`"(.*)`"" /commit:apphost
appcmd set config -section:system.webServer/rewrite/rules "/+[name='HTTPS_301_Redirect',stopProcessing='False'].conditions.[input='{HTTPS}',pattern='off']" /commit:apphost
appcmd set config -section:system.webServer/rewrite/rules "/[name='HTTPS_301_Redirect',stopProcessing='False'].action.type:`"Redirect`"" "/[name='HTTPS_301_Redirect',stopProcessing='False'].action.url:`"https://{HTTP_HOST}{REQUEST_URI}`"" /commit:apphost
{% endhighlight %}

## Implement HTTP Strict Transport Security (HSTS)

One new standard making its way across the web is HTTP Strict Transport Security. The gist of it is that once a browser has been to a website with HTTPS, we want the browser to continue going to that website with HTTPS and not try to downgrade to HTTP.

Again, we will be referencing OWASP, but here is the script.

{% highlight powershell %}
appcmd set config -section:system.webServer/rewrite/outboundRules /+"preConditions.[name='USING_HTTPS']" /commit:apphost
 & $($env:windir + "\system32\inetsrv\appcmd.exe") set config -section:system.webServer/rewrite/outboundRules /"+preConditions.[name='USING_HTTPS'].[input='{HTTPS}',pattern='on']" /commit:apphost
appcmd set config -section:system.webServer/rewrite/outboundRules /+"[name='Add_HSTS_Header',preCondition='USING_HTTPS']" /commit:apphost
appcmd set config -section:system.webServer/rewrite/outboundRules "/[name='Add_HSTS_Header'].patternSyntax:`"Wildcard`"" /commit:apphost
appcmd set config -section:system.webServer/rewrite/outboundRules "/[name='Add_HSTS_Header',preCondition='USING_HTTPS'].match.serverVariable:`"RESPONSE_Strict-Transport-Security`"" "/[name='Add_HSTS_Header',preCondition='USING_HTTPS'].match.pattern:`"*`"" /commit:apphost
appcmd set config -section:system.webServer/rewrite/outboundRules "/[name='Add_HSTS_Header',preCondition='USING_HTTPS'].action.type:`"Rewrite`"" "/[name='Add_HSTS_Header',preCondition='USING_HTTPS'].action.value:`"max-age=31536000`"" /commit:apphost
{% endhighlight %}

## Remove Server Identification

This is more of a wishlist item than a necessity.

In WWII, they said loose lips sink ships.

Microsoft Internet Information Services (IIS) is really noisy by default. It tells the world “hey, this is an IIS server powered by ASP.NET.” This isn’t necessarily bad, but it isn’t great either. The less information we can give the attackers about what they are about to attack, the better.

The problem with IIS is that it’s actually somewhat difficult to actually remove the Server header that displays what type of server the application is running on. You can scrub the [Server response variable in IIS](https://stackoverflow.com/questions/1178831/remove-server-response-header-iis7/12615970#12615970), but the variable is still there.

The other options are to install [URL Scan](http://www.iis.net/downloads/microsoft/urlscan) on your web server, but this isn’t compatible with IIS8 on Server 2012, and it can be somewhat temperamental with requests if the web developer hasn’t ensured that the application inputs are sanitized correctly.

Another option would be to [remove the Server response header](http://blogs.technet.com/b/stefan_gossner/archive/2008/03/12/iis-7-how-to-send-a-custom-server-http-header.aspx) in the application itself.

For now, I will stick to the URL rewrite rule removing the server version. Let’s also remove the “Powered By ASP.NET” header.

{% highlight powershell %}
appcmd set config -section:system.webServer/rewrite/outboundRules /+"[name='Remove_RESPONSE_Server']" /commit:apphost
appcmd set config -section:system.webServer/rewrite/outboundRules "/[name='Remove_RESPONSE_Server'].patternSyntax:`"Wildcard`"" /commit:apphost
appcmd set config -section:system.webServer/rewrite/outboundRules "/[name='Remove_RESPONSE_Server'].match.serverVariable:'RESPONSE_Server'" "/[name='Remove_RESPONSE_Server'].match.pattern:`"*`"" /commit:apphost
appcmd set config -section:system.webServer/rewrite/outboundRules "/[name='Remove_RESPONSE_Server'].action.type:`"Rewrite`"" "/[name='Remove_RESPONSE_Server'].action.value:`" `"" /commit:apphost
appcmd set config /section:httpProtocol "/-customHeaders.[name='X-Powered-By']"
{% endhighlight %}

## Prevent clickjacking

Clickjacking (or [framejacking](https://www.owasp.org/index.php/Clickjacking)) is a way for attackers to make people think they are on a different website and trick them into giving credentials to access the real website. An embedded frame may appear to be on your Facebook login, but in reality, you’re typing your credentials somewhere else.

To prevent this, we will use [Microsoft guidance](https://support.microsoft.com/en-us/kb/2694329).

{% highlight powershell %}
appcmd set config -section:httpProtocol "/+customHeaders.[name='X-Frame-Options',value='SAMEORIGIN']"
{% endhighlight %}

## Putting it all together

The script I put together to mitigate these issues can be found on [Github](https://github.com/drewhjelm/iis-hardening).

## Running the Script

I spun up a fresh Windows 2012 R2 server, installed IIS 8.5 and an SSL cert from Comodo then ran an SSL Labs test. Here is the result:

![Initial Test]({{ site.url }}/images/ssl-hardening/ssl-test-begin.png)

A fresh server is definitely not ready to handle your secure data yet.  

![Initial Test]({{ site.url }}/images/ssl-hardening/ssl-begin-protocol-and-ciphers.png)

![Initial Test]({{ site.url }}/images/ssl-hardening/ssl-begin-protocol-details.png)

To run the script on a server, you’ll need to install:

Your SSL Certificate
URL Rewrite 2.0
Open a PowerShell Window and import the script

{% highlight powershell %}
PS C:\users\user\Desktop\ '.\configure IIS Security.ps1'
{% endhighlight %}

When you import the script, you may see a warning about script execution policy.

![PowerShell Permissions]({{ site.url }}/images/ssl-hardening/powershell-permissions.png)

After enabling the Execution Policy change, run the script with:

{% highlight powershell %}
Set-IISSecurity
{% endhighlight %}

The output should be similar to this:

![Script Run]({{ site.url }}/images/ssl-hardening/script-run.png)

And a prompt to restart the server will appear.

After restarting the server, rerun the SSL Labs test and you should see something like this:

![Hardened Test]({{ site.url }}/images/ssl-hardening/ssl-test-after.png)

![Hardened Test]({{ site.url }}/images/ssl-hardening/ssl-after-protocol-details.png)

## Notes in the margin:

* You will be unable to get an A+ rating on SSL labs with IIS because [Microsoft will not implement TLS_FALLBACK_SCSV](https://connect.microsoft.com/IE/feedback/details/1002874/internet-explorer-should-send-tls-fallback-scsv) until the version of Windows Server after 2012 R2. You could theoretically disable TLS 1.0 and 1.1 to prevent a fallback attack (like POODLE), but in doing that you would break compatibility with many older browsers.
* This script breaks compatibility with IE 6 on Windows XP and Java 6 Update 45 clients.
* If your web server communicates with other servers through APIs, you may need to tweak your cipher suites to be compatible.
* This script does not protect against application vulnerabilities, such as SQL Injection, cross-site scripting (XSS), or cross-site request forgery (CSRF) vulnerabilities. Always consult with your application developers to ensure they are following proper secure development procedures.
* This script does not assist with configuring your application securely. When deploying your application, you will need to follow best practices such as creating a new application pool for each application on the server to isolate processes, limiting the user accounts’ permissions running the application pool, and separating your data and application tiers.