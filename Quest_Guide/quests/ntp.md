---
title: NTP
layout: default
---

# NTP

## Quest objectives

- Use the `puppet module` tool to find and install modules on the Puppet Forge
- Learn how you can use the `site.pp` manifest to classify nodes.
- Use class parameters to adjust variables in a class as you declare it.

## Getting started

In the Modules quest, you learned about the structure of a Puppet module and
how to create one. It's important to have some hands-on module-writing
experience so you know what you're doing when you integrate existing code
into your infrastructure. It's just as important, however, that you
understand how to make use of existing modules. Using an existing module
isn't just easier. When you use a publicly available module, you're often
getting code that has already been tested and deployed across hundreds or
thousands of other users' infrastructures.

Furthermore, using [Puppet Supported](https://forge.puppetlabs.com/supported)
and [Puppet Approved](https://forge.puppetlabs.com/approved) modules
adds another layer of validation and reliability.

Keep in mind, though, that no matter whose code you're using, relying on external
checks is no substitute for your own thorough review and testing
of anything you're putting into production!

In this quest, you'll learn how you can use an existing
module from the Puppet Forge to manage an important service on your
machine: NTP.

    quest --start ntp

## What's NTP

> Time is the substance from which I am made. Time is a river which carries me
> along, but I am the river; it is a tiger that devours me, but I am the tiger;
> it is a fire that consumes me, but I am the fire.

> -Jorge Luis Borges

Security services, shared filesystems, certificate signing, logging systems, and
many other fundamental services and applications (including Puppet itself!) need
accurate and coordinated time to function reliably. Given variable network
latency, it takes some clever algorithms and protocols to get this coordination
right.

The Network Time Protocol (NTP) lets you keep time millisecond-accurate within
your network while staying synchronized to Coordinated Universal Time (UTC) by
way of publicly accessible timeservers. (If you're interested in the subtleties
of how NTP works, you can read all about it
[here](http://www.eecis.udel.edu/~ntp/ntpfaq/NTP-a-faq.htm))

NTP is one of the most fundamental services you will want to include in your
infrastructure. Puppet Labs maintains a supported module that makes the
configuration and management of NTP simple.

## Package/File/Service

We'll show you how to install and deploy the NTP module in a moment, but first,
take a look at the current state of your system. This way, you'll be able to
keep track of what Puppet changes and understand why the NTP module does what it
does.

To get the NTP service running, there are three key resources that Puppet will
manage. The `puppet resource` tool can show you the current state of each of
these resources.

First, check the state of the NTP *package*:

    puppet resource package ntp
	
check the NTP configuration *file*:
	
    puppet resource file /etc/ntp.conf

finally, see if the Network Time Protocol Daemon (NTPD) *service* is running:

    puppet resource service ntpd
	
You'll see that the NTP package is installed on the Learning VM, that the
configuration file exists, but that the ntpd service is 'stopped'.

As you continue to work with Puppet, you'll find that this
*package/file/service* pattern is very common. These three resource types
correspond to the common sequence of installing a package, customizing that
package's functionality with configuration files, and starting the service
provided by that package.

The *package/file/service* pattern also describes the typical relationships of
dependency among these three resources: a well-written class will define these
relationships, telling Puppet to restart the service if the configuration file
has been modified, and re-create the configuration file when the package is
installed or updated. You'll be working with an existing module in this quest,
so these dependencies are already taken care of; we'll get into the specifics of
how they can be managed in a later quest.

But now, on to the installation!

## Installation

Before you classify the Learning VM with the NTP class, you'll need to install
the NTP module from the Forge. While the module itself is called `ntp`, recall
that modules in the Forge are prefixed by the account name of the associated user.
So to get the Puppet Labs NTP module, you'll specify `puppetlabs-ntp`. When you
look at the module saved to the modulepath on your Puppet master, however, it will
be named `ntp`. Keep this in mind, as trying to install multiple modules of the
same name can lead to conflicts!

{% task 1 %}
---
- execute: puppet module install puppetlabs-ntp
{% endtask %}

Use the Puppet module tool to install the Puppet Labs `ntp` module.

    puppet module install puppetlabs-ntp
	
This command tells the Puppet module tool to fetch the module from the Puppet
Forge and place it in Puppet's modulepath:
`/etc/puppetlabs/code/environments/production/modules`.

## Classification with the site.pp manifest

Now that the NTP module is installed, all the included classes are available to
use in node classification.

In the Power of Puppet quest, you learned how to classify a node with the PE
Console. In this quest, we introduce another method of node classification: the
`site.pp` manifest.

`site.pp` is the first manifest the Puppet agent checks when it connects to the
master. It defines global settings and resource defaults that will apply to all
nodes in your infrastructure. It is also where you will put your *node
definitions* (sometimes called `node statements`).

A node definition is the code-defined equivalent of the node group you saw in
the Power of Puppet quest.

{% highlight puppet %}
node 'learning.puppetlabs.vm' {
  ...
}
{% endhighlight %}

{% aside PE console or site.pp? %}
Because it's more amenable to monitoring with the Learning VM quest tool, we'll
be primarily using this `site.pp` method of classification in this Quest Guide.
What you will learn about node definitions and class declarations applies to whatever
methods of classification you decide to use later, including the PE Console node
classifier.
{% endaside %}

{% task 2 %}
---
- execute: vim /etc/puppetlabs/code/environments/production/manifests/site.pp
  input:
    - "/default {\r"
    - o
    - "include ntp"
    - "\e"
    - ":"
    - "wq\r"
{% endtask %}

Open the site.pp manifest in your text editor.

    vim /etc/puppetlabs/code/environments/production/manifests/site.pp
	
Skip to the bottom of the file. (You can use the vim shortcut `G`)

You'll see a `default` node definition. This is a special node definition that
Puppet will apply to any node that's not specifically included in any other node
definition.

We only want our changes to apply to the Learning VM, however, so we'll put our
`ntp` class declaration in a new `learning.puppetlabs.vm` node block.

{% highlight puppet %}

node 'learning.puppetlabs.vm' {
  include ntp
}

{% endhighlight %}

{% task 3 %}
---
- execute: puppet agent -t
{% endtask %}

Note that triggering a Puppet run with the `puppet agent` tool is useful for
learning and testing, but that in a production environment you would want to
let the Puppet agent run as scheduled, every 30 minutes, by default. Because
you'll be running Puppet right after making changes to the `site.pp` manifest,
Puppet may not have a chance to refresh its cache. If your changes to the
`site.pp` manifest aren't reflected in a Puppet run triggered by the
`puppet agent -t` command, try running the command again.

Test the `site.pp` manifest with the `puppet parser validate` command, and trigger
a Puppet run.

    puppet agent -t

Once the Puppet run is complete, use the Puppet resource tool to inspect the
`ntpd` service again. If the class has been successfully applied, you will see
that the service is running.

### Syncing up

To avoid disrupting processes that rely on consistent timing, the ntpd service
works gradually. It adds or removes a few microseconds to each tick of the
system clock so as to slowly bring it into synchronization with the NTP server. 

If you like, run the `ntpstat` command to check on the synchronization status.
Don't worry about waiting to get synchronized. Because the Learning VM is
virtual, its clock will probably be set based on the time it was created or last
suspended. It's likely to be massively out of date with the time server, and it
may take half an hour or more to get synchronized!

## Class defaults and class parameters

The `ntp` class includes default settings for most of its parameters. The
`include` syntax you used let you concisely declare the class without modifying
these defaults.

One of these defaults, for instance, tells Puppet which time servers to include
in the NTP configuration file. To see what servers were specified by default,
you can check the configuration file directly. Enter the command:
	
    grep server /etc/ntp.conf
	
You'll see a list of the default servers:

    server 0.centos.pool.ntp.org
    server 1.centos.pool.ntp.org
    server 2.centos.pool.ntp.org

These ntp.org servers aren't actually time servers themselves; rather, they're
access points that will pass you on to one of a pool of public timeservers. Most
servers assigned through the ntp.org pool are provided by volunteers running NTP
as an extra service on a mail or web server. 

While these work well enough, you'll get more accurate time and use less network
resources if you pick public timeservers in your area.

To manually specify which timeservers your NTPD service will poll, you'll need
to override the default ntp.org pool servers set by the NTP module.

This is where Puppet's *class parameters* come in. *Class parameters* provide a
method to set variables in a class as it's declared. The syntax for
parameterized classes looks similar to the syntax for resource declarations.
Have a look at the following example:

{% highlight puppet %}

class { 'ntp':
  servers => [
    'nist-time-server.eoni.com',
    'nist1-lv.ustiming.org',
    'ntp-nist.ldsbc.edu'
  ]
}

{% endhighlight %}

The `servers` parameter in our class declaration takes a list of servers as a
value, not just one. This list of values, separated by commas (`,`) and wrapped
in brackets (`[]`), is called an *array*. Arrays allow you assign a list of
values to a single variable or attribute.

{% task 4 %}
---
- execute: vim /etc/puppetlabs/code/environments/production/manifests/site.pp
  input:
    - "/include ntp\r"
    - dd
    - i
    - "  class { 'ntp':\r"
    - "servers => ['nist-time-server.eoni.com','nist1-lv.ustiming.org','ntp-nist.ldsbc.edu']\r"
    - "}"
    - "\e"
    - ":"
    - "wq\r"
{% endtask %}

In your `site.pp`, replace the `include ntp` line with a parameterized class
declaration based on the example above. Use the servers from the example, or, if
you know of a nearer timeserver, include that. You should always specify at
least *three* timeservers for NTP to function reliably. You might, for instance,
include two from the ntp.org pool and one known nearby timeserver.

{% task 5 %}
---
- execute: puppet agent -t
{% endtask %}

Once you've made your changes to the `site.pp` manifest and used the `puppet parser`
tool to validate your syntax, use the `puppet agent` tool to trigger a
Puppet run.

You will see in the output that Puppet has changed the `/etc/ntp.conf` file and
triggered a refresh of the `ntpd` service.

## Review

We covered some details of finding and downloading modules from the Puppet Forge
with the `puppet module` tool. We also covered the common Package/File/Service
pattern, and how it's used by the NTP module to install, configure, and run the
ntpd service.

Rather than just running tests, you learned how to use the `site.pp` manifest to
include classes within a node declaration.

After getting the ntpd service running, we went over class parameters, and
showed how they can be used to set class parameters as a class is declared.
