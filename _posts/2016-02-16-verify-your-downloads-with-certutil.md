---
layout: post
title: Verify your downloads with Certutil
---

Downloading files from the internet can be dangerous, since some less-than-reputable sites can inject files into your chosen download and put something in there you don’t want. In order to ensure you are downloading and running the files you intended on grabbing, you should verify the integrity of the download. Integrity is a security service that ensures data has not been tampered or altered by unauthorized parties.

A file’s integrity can be checked using a hash, which is an algorithm that takes all information in a file and turns it into a non-reversible string. Any time a file is changed, the value of the hash changes.

For instance, if we save a file named hash.txt with the text “this is a hash” the SHA1 hash value is

{% highlight powershell %}
b6 fe a4 73 62 3c d1 47 4c 8a d2 1c b2 23 be de 76 6e 24 dc
{% endhighlight %}

where changing the text to “this is hash” changes the SHA1 hash value to


{% highlight powershell %}
54 af 3f f9 4b a1 1a c9 3b 46 82 7e d5 e7 e1 d8 ae d5 53 4a
{% endhighlight %}

One letter will completely change the hash value, and so we know with certainty that something has changed.

Let’s take a look at this in action. Say you’re going to download [Kali Linux](https://www.kali.org/), a powerful distro of Linux tailored for penetration testing and hacking, and you get to the [download page](https://www.kali.org/downloads/):

![Kali Hash]({{ site.url }}/images/certutil/kali-hash-768x323.png)

If you’re using Windows, you can use Certutil to verify the hash of the file. Windows 8 and newer have Certutil pre-installed. Open up a command prompt (or Powershell prompt, your choice) and type:

{% highlight powershell %}
certutil -hashfile <path-to-file> SHA1
{% endhighlight %}

In this case, the file was exactly what we expected when downloaded. I downloaded the 64-bit version of Kali:

![certutil]({{ site.url }}/images/certutil/certutil.png)

The ISO has the same hash value and we can assume this is ISO hasn’t been tampered.

The hashing algorithm in this case is SHA1, but sometimes MD5 is also used. MD5 is not safe for cryptographic purposesbecause as a 128-bit hash it is more susceptible to collisions where two items will have the same hash value, but some sites will supply the MD5 hashes for files available for download in addition to the SHA1 hash.

File hashing has other applications as well. In addition to assisting with ensuring that downloaded files from the internet have not been altered, services like host-based IDS and file integrity monitors like TripWire can use file hashes to verify that key system files haven’t been altered and alert you in the event of unusual file alteration.