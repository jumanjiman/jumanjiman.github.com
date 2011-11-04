---
layout: post
title: "Check boot-time service status with systemctl"
date: 2011-11-04 18:29
comments: true
categories: systemctl
---
Seth Vidal posted a useful snippet at [http://skvidal.wordpress.com/2011/11/04/like-chkconfig-list](http://skvidal.wordpress.com/2011/11/04/like-chkconfig-list)
today for checking the boot-time status of services on a host
using [systemd](http://fedoraproject.org/wiki/SysVinit_to_Systemd_Cheatsheet).

I copied and pasted from his blog into my shell, only
to discover that Wordpress had transformed his double-quotes
into smart quotes. Bleh. So I hacked for a couple minutes
and came up with this [new version](https://gist.github.com/1340655):

{% gist 1340655 %}

Here's 10 lines of output from the script:

``` text Boot-time service status
abrtd.service                                             on
accounts-daemon.service                                  off
acpid.service                                             on
atd.service                                               on
atop.service                                             off
avahi-daemon.service                                      on
bluetooth.service                                         on
canberra-system-bootup.service                           off
canberra-system-shutdown-reboot.service                  off
canberra-system-shutdown.service                         off
```

Enjoy!
