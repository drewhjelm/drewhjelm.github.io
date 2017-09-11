---
layout: post
title: Alienvault warning “asset logs are not being processed”
---

After running an update on [Alienvault OSSIM](https://www.alienvault.com/open-threat-exchange/projects) and adding new subnets to an existing deployment, I kept getting warnings that the  Sonicwall firewall was sending logs to the Alienvault appliance but the data source plugin needed to be configured. The exact warning for the firewall: Asset logs are not being processed.”

Curious, I looked at the recent SIEM events and saw that events on the Sonicwall firewall were being processed, so I thought something was wrong with the Alienvault. I confirmed that the [Sonicwall configuration](https://alienvault.bloomfire.com/posts/596832-device-integration-sonicwall/public) was correct.

I then noticed that the assets on the new subnet were not in the inventory, so I re-scanned the network and updated the inventory. After the inventory was updated, the warning was gone.