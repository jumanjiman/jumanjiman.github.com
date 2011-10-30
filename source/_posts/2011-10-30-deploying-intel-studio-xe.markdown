---
layout: post
title: "Deploying Intel Studio XE 2011"
date: 2011-10-30 14:19
comments: true
categories: ['rpm', 'vendor software', 'deployment']
---
Intel uses the Macrovision floating license manager (flexlm)
to prevent you from using its developer products without authorization.
You need flexlm on one server if you are using floating licenses
for Intel client products, such as VTune or icc.
Flexlm running on Linux provides support for both Windows and
Linux clients running Intel studio.

If you want things to "just work", you have some work to do.
For starters, flexlm is shipped as a tarball with an interactive
install script. Worse, the documentation for flexlm is
spotty and fails to provide a useful init script.
There is always more that one way to do it, but this post
shows one approach to bundling and deploying a third-party
application in a repeatable fashion.

In this post, I provide some tips for deploying the flex license
manager for Intel developer tools on RHEL 5 (64-bit), including:

* Prepare the license file
* Package flexlm as an RPM
* Create a SysV-style init script for flexlm that works
  with Red Hat Cluster Suite (RHCS)

<!-- more -->

## Setup the floating license file ##

### Combine license files into a single file ###

Flexlm will not start without a valid license file, so
our first step is to create a file which combines
one or more licenses.

``` text /opt/intel/licenses/server.lic
SERVER id-lic01 00174F0D0060 28518
VENDOR INTEL port=28519
PACKAGE I3FD67AA5 INTEL 2011.1221 8BC9DA9CDC44 COMPONENTS="ArBBL \
...
INCREMENT F4AC67AA5 INTEL 2011.1221 21-dec-2011 5 CAAAAAAAAAA1 \
...
INCREMENT I3FD67AA5 INTEL 2011.1221 21-dec-2011 5 FAAAAAAAAAA1 \
...
```

Important: _The license file also configures the network ports
for the license server._
In the above snippet, line 1 tells the master flexlm server to run on port 28518,
while line 2 tells the vendor server (INTEL) to run on port 28519.
By default, the vendor port is missing from the license file.
You can manually add the port to the vendor line without
compromising the signature in the license.

Line 3 is the first individual license, and lines 5 & 7 indicate
the start of additional licenses. The SERVER and VENDOR lines
should appear only once in your license file.
Naturally, I changed the private information in the above snippet
so nobody can steal our license or complain about unauthorized sharing.

### Export an environment variable ###

Every host needing to run the Intel tools with a floating license
needs to know where the license file lives. To keep things simple,
deploy a file `/etc/profile.d/intel-licens.sh`:

``` bash /etc/profile.d/intel-license.sh
if [ -n "$BASH_VERSION" -o -n "$KSH_VERSION" -o -n "$ZSH_VERSION" ]; then
  # export an environment variable for Intel floating licenses
  INTEL_LICENSE_FILE=${INTEL_LICENSE_FILE:-"/opt/intel/licenses/server.lic"}
  export INTEL_LICENSE_FILE
fi
```

In the above snippet, line 3 provides a default path to our
system-wide license file but does _not_ override a user's
personal license file, if any.

### Bundle the license file and profile script as an RPM ###

You can deploy the license file and profile script however
you see fit. You can deploy directly with [Puppet](http://puppetlabs.com/),
[Chef](http://wiki.opscode.com/display/chef/Home),
[bcfg2](http://trac.mcs.anl.gov/projects/bcfg2/),
or any other mechanism. You can also bundle the above two
files as an RPM using a spec file like this, which you can
[download or clone](https://gist.github.com/1326195) from GitHub:

{% gist 1326195 %}

In the above spec file, lines 45-53 provide an rpm scriptlet
that runs after installation. Line 46 uses `sed` to delete
our modifications to `/etc/hosts`. Lines 47-53 add an entry
to `/etc/hosts` to guarantee that each client can resolve
the name of our license server in case of DNS misconfiguration.

## Package flexlm as an RPM ##

### Initial setup for flexlm ###

First, extract the flexlm tarball and run `Install_INTEL` on a
throw-away virtual machine. The script creates several files
and some symbolic links. We're not going to use this tree
directly. Instead, we'll create a separate directory and copy
some of the files into an alternate tree.

* A top-level `flexlm-server` directory to store all the necessary files
* A `docs` directory to store the installation log
  and config file from the interactive installation
* A `flexlm` directory for proprietary files from the installation
* A `src` directory for my own SysV-style init script,
  which I'll provide later in this post

In my case, I created four directories and copied a subset
of the files into this alternate tree:

    flexlm-server/
    ├── docs
    │   ├── Install_INTEL.cfg
    │   └── Install_INTEL.log 
    ├── flexlm
    │   ├── chklic
    │   ├── END_USER_LICENSE
    │   ├── enduser.pdf
    │   ├── getip
    │   ├── HowTo.html
    │   ├── Install_INTEL
    │   ├── INTEL
    │   ├── lmgrd.intel
    │   ├── lmutil
    │   └── README
    └── src

### Create a SysV-style init script for flexlm ###

In modern init scripts, the exit code must *not* be based
on the state *transition*. According to 
[LSB requirements](http://refspecs.freestandards.org/LSB_3.1.1/LSB-Core-generic/LSB-Core-generic/iniscrptact.html),
the exit code must be based on the final *state* of the service:

* Upon a `start`, the script must return zero if
  the service is running normally
  and a non-zero exit code otherwise
* Upon a `stop`, the script must return zero if the service
  is not running and a non-zero exit code otherwise
* Upon a `restart`, the script must return zero if the service
  is running normally after the script completes
  and a non-zero exit code otherwise
* Upon a `status`, the script must return zero if the service
  is running normally and must return non-zero otherwise

Intel recommends using `lmstat` to confirm status.
Unfortunately, the init script cannot use the exit code of `lmstat`
to determine status.
For example, consider the following run of `./lmstat`,
in which the flexlm server is *not* running:

``` bash Output of lmstat when server is not running
lmstat - Copyright (c) 1989-2004 by Macrovision Corporation. All rights reserved.
Flexible License Manager status on Fri 3/4/2011 12:22

 License server status: 28518@ic-lic01
    License file(s) on ic-lic01: /usr/local/flexlm/licenses/license.dat:@id-lic01:server.lic:

lmgrd is not running: Server node is down or not responding (-96,7)
```

The output clearly indicates that the server is down, but
the exit status indicates the server is running:

``` bash Wrong exit status when server is not running
echo $?
0
```

All is not lost.
The exit code of `lmstat` is untrustworthy for our init script,
but we can `grep` the output for key phrases.
Here's the output of `lmstat` when the service _is_ running:

``` bash Output of lmstat when server is running normally
lmstat - Copyright (c) 1989-2004 by Macrovision Corporation. All rights reserved.
Flexible License Manager status on Fri 3/4/2011 12:22

License server status: 28518@id-lic01
    License file(s) on id-lic01: /opt/intel/flexlm/server.lic:

  id-lic01: license server UP (MASTER) v9.2

Vendor daemon status (on id-lic01):

     INTEL: UP v9.2
```

Before writitng the init script, we need to understand that
there are actually *two* services to check.
The first is  the `lmgrd` master daemon, and
the second is the `INTEL` vendor daemon.

Without further ado, let me just show the init script I came up with.
You can [clone or download it from GitHub](https://gist.github.com/1296847):

{% gist 1296847 %}

### Make a spec file ###

There are numerous references for building spec files.
I try to adhere to the
[Fedora Packaging Guidelines](http://fedoraproject.org/wiki/Packaging:Guidelines)
when possible.

This RPM includes proprietary code and could
never be considered for inclusion in any free distro, but
the spec file works for me in a private deployment.
The spec file itself is mine and is distributed under the GPLv2.
You can [download or clone](https://gist.github.com/1325001)
this spec file from GitHub.
Put this spec file in the top level of your flexlm-server directory.

{% gist 1325001 %}

Lines 98-102 of the spec file define an rpm scriptlet that
runs _before_ the rpm is uninstalled. In line 99, we test
the `$1` variable to see if this is the last removal of the package
and, if so, stop the service and remove the init symlinks before
actually uninstalling the package. We use a bash noop (`:`)
to ensure the rpm scriptlet does not fail. If a scriptlet fails,
the package will not be uninstalled.

Lines 105-111 define an rpm scriptlet that runs _after_ the
rpm has been installed. Line 106 checks to ensure that we
have at least one copy of the rpm installed. If so, line 107
ensures the SysV init symlinks exist without changing the
boot-time startup. Lines 108-110 check to see if the service
is currently running. If it _is_ running, we restart the
service to pick up any changes from this version of the package.
Again, we use a noop to prevent the scriptlet from failing
during an upgrade or fresh installation. If the service fails
for any reason, such as broken networking, we'll still have
the package in place for troubleshooting.

### Build the RPM ###

I use [tito](https://github.com/dgoodwin/tito) for most of my
rpm-building needs.
It's available in the Fedora and [EPEL](http://fedoraproject.org/wiki/EPEL)
yum repos.

We have our directory tree setup, so now we initialize it for tito:

``` bash Build with tito
tito init
tito tag --keep-version
tito build --rpm
```

Assuming all goes well, you'll have a source rpm and a binary rpm,
ready to be added to an in-house yum repo for deployment.
