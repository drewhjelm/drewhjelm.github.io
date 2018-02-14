---
layout: post
title: Parse FortiGate rules into CSV using PowerShell
---

FortiGate firewalls built by Fortinet provide organizations with a robust next-generation firewall platform to secure networks with Unified Threat Management (UTM) features. I routinely perform audits on FortiGate firewalls. The syntax is not difficult to read, but when a larger device ends up with 100+ rules it can be cumbersome to try and read all of them at once.

tl;dr: the script is on [Github](https://github.com/drewhjelm/firewall-audit).

There are a few things I look for in firewall rules, including:

* Number of rules with ALL allowed in interfaces and/or addresses - Looking through rules with permissive ingress/egress rules helps see where an organization is not properly limiting its traffic. Helping organizations move toward a [zero-trust architecture](https://www.paloaltonetworks.com/cyberpedia/what-is-a-zero-trust-architecture) is an important goal.
* Number of rules with ALL allowed in services - While application-aware rules on the FortiGate can help to limit services on rules appropriately, rules with ALL services allowed can be a sign that [least functionality principles](https://nvd.nist.gov/800-53/Rev4/control/CM-7) may not have been considered in a network. The PCI DSS specifically requires documentation of business justification for allowed ports, protocols, and services, and requires insecure protocols to not be used for sensitive or administrative purposes.
* Rules with DNS and NTP egress allowed - Secure DNS architecture in an organization requires that DNS queries are being monitored and logged appropriately. Devices should not be allowed to make external DNS queries without passing through the organization's internal DNS servers. Organizations need to also ensure that they are synchronizing their devices to the same internal NTP sources in order to make sure that events can be correlated during an incident investigation. DNS and NTP are also being used as methods of command and control for adversaries.
* Number of rules with UTM features enabled - Organizations should ensure they are taking advantage of the full set of features in a FortiGate firewall, including antivirus, application signatures, and intrusion protection (IPS) functions. 
* Number of rules with names and comments - Organizations that aren't properly naming and tracking the rationale for rules being enabled run the risk of having configuration sprawl. Regular review of firewall rules and the justification for these rules existing is important for configuration management. Pointing out the rules without names or comments helps show an organization where they may not have appropriately documented a change they made.

In addition to different items to find in firewall rules, I also wanted to make it easier to filter various config files I started reviewing to determine how the devices are being configured. 

Turning the rules in a FortiGate configuration file into a CSV that can be easily reviewed in Excel seemed to be the best way to handle these requirements. As I work primarily in Windows, the preferred scripting language of choice is PowerShell. 

FortiGate configs do not easily parse because they are in a text format rather than a markup language like XML or JSON, so the first step to building a config parser is to figure out where the rules start and then start looking at each subsequent rule for patterns. A small rule set in a FortiGate config file might look like:

{% highlight shell-session %}
config firewall policy
    edit 1
        set name "local -> internet"
        set uuid bcf9e79e-fe21-51e7-a8fa-4e5ac3446336
        set srcintf "interface1"
        set dstintf "wan1"
        set srcaddr "LAN-PCs"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "HTTP" "HTTPS"
        set utm-status enable
        set nat enable
    next
    edit 2
        set name "Management -> AD DC"
        set uuid 27ece351-ee23-42e6-9963-9a5e6d477cd5
        set srcintf "interface1"
        set dstintf "interface2"
        set srcaddr "MGMT-PC"
        set dstaddr "DC"
        set action accept
        set schedule "always"
        set service "RDP"
    next
    edit 3
        set name "LAN -> internal-DNS-AD"
        set uuid e5188148-c503-4da7-865e-c26f8b29f30c
        set srcintf "interface1"
        set dstintf "interface2"
        set srcaddr "LAN PCs"
        set dstaddr "DC"
        set action accept
        set schedule "always"
        set service "LDAPS" "DNS"
    next
    edit 4  
        set name "DC -> outbound-DNS"
        set uuid 8506af4b-b701-48ac-8b69-ee8ee2b5870e
        set srcintf "interface2"
        set dstintf "wan1"
        set srcaddr "DC"
        set dstaddr "Google DNS" "Quad 9"
        set action accept
        set schedule "always"
        set service "DNS"
        set nat enable
    next
end
{% endhighlight %}

So within the rule, there are several items we can use to parse the rule set:

* The `config firewall policy` indicates the beginning of the rule set in the config file
* Each rule will start `edit n`, where _n_ is the rule number. This can function as the ID of a rule in our CSV.
* Each rule property will start `set` and define some property (IP addresses, service, etc.)
* Each rule will end `next` and we can then look at the next rule
* The rule set stops at `end`.

One thing I noticed is that some rules will not define all properties. Some may have default actions, and other versions of FortiOS may not support a particular feature. To accommodate this, I have built a dummy rule that includes all the properties I could find. If a rule set is parsed that doesn't begin with all the properties, then only those properties in the first rule would be defined in the CSV and the rest would be dropped. I attempted to build custom PowerShell objects to handle this, but had difficulty and ended up creating a standalone text file to be added into the rule during parsing.

{% highlight shell-session %}
    edit "deleteme"
        set name "deleteme"
        set uuid "deleteme"
        set srcintf "deleteme"
        set dstintf "deleteme"
        set srcaddr "deleteme"
        set dstaddr "deleteme"
        set action "deleteme"
        set schedule "deleteme"
        set service "deleteme"
        set utm-status "deleteme"
        set logtraffic "deleteme"
        set av-profile "deleteme"
        set ips-sensor "deleteme"
        set profile-protocol-options "deleteme"
        set ssl-ssh-profile "deleteme"
        set application-list "deleteme"
        set nat "deleteme"
        set status "deleteme"
        set webfilter-profile "deleteme"
        set poolname "deleteme"
        set comments "deleteme"
    next
{% endhighlight %}

The script would then do the following:

1. Load the config
2. Identify the [VDOM](http://cookbook.fortinet.com/vdom-configuration/) it was reviewing
3. Find the beginning of the VDOM rule set
4. Add a dummy rule to the beginning of the rule set
5. Parse through the rules in the rule set
6. Create a CSV for each VDOM.

PowerShell script is on [Github](https://github.com/drewhjelm/firewall-audit)

To run from command line:

{% highlight powershell %}
./Parse-FortiGateRules.ps1 -fortiGateConfig "c:\temp\config.conf"
{% endhighlight %}

This would create a csv of the config.conf rules in c:\temp

To quickly run through multiple config files:

{% highlight powershell %}
Get-ChildItem -Filter *.conf | Foreach-Object { ./Parse-FortiGateRules.ps1 -fortiGateConfig $_.FullName }
{% endhighlight %}

Any questions or comments feel free to create an issue on Github or tweet @drewhjelm.