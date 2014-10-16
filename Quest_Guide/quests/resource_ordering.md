---
title: Ordering
layout: default
---

# Resource Ordering

### Prerequisites

- Begin Quest
- Welcome Quest
- Power of Puppet Quest
- Resources Quest
- Manifest Quest
- Variables Quest
- Conditions Quest

## Quest Objectives

 - Understand why some resources must be managed in a specific order.
 - Use the `before`, `require`, `notify`, and `subscribe` metaparameters to effectively manage the order that Puppet applies resource declarations.

## Getting Started

This quest will help you learn more about specifying the order in which Puppet should manage resources in a manifest. When you're ready to get started, type the following command:

	quest --start ordering

## Explicit Ordering

We are likely to read instructions from top to bottom and execute them in that order. When it comes to resource declarations in a Puppet manifest, Puppet does things a little differently. It works through the problem as though it were given a list of things to do, and it was left to decide the most efficient way to get them done. We have referred to the catalog vaguely in the previous sections. The **catalog** is a compilation of all the resources that will be applied to a given system, and the relationships between those resources. In building the catalog, unless we _explicitly_ specify the relationship between the resources, Puppet will manage them in its own order.  

For the most part, Puppet specifies relationships between resources in the appropriate manner while building the catalog. For example, if you say that user `gigabyte` should exist, and the directory `/home/gigabyte/bin` should be present and be owned by user `gigabyte`, then Puppet will specify a relationship between the two - that the user should be managed before the directory. These are implicit (shall we call them obvious?) relationships. 

Sometimes, however, you will need to ensure that a resource declaration is applied before another. For instance, if you wish to declare that a service should be running, you need to ensure that the package for that service is installed and configured before you can start the service. One might ask as to why there is not an implicit relationship in this case. The answer is that, often times, more than one package provides the same service, and what if you are using a package you built yourself? Since Puppet cannot _always_ conclusively determine the mapping between a package and a service (the names of the software package and the service or executable it provides are not always the same either), it is up to us to specify the relationship between them.

When you need a group of resources to be managed in a specific order, you must explicitly state the dependency relationships between these resources within the resource declarations.

## Relationship Metaparameters

{% fact %}
A metaparameter is a resource attribute that can be specified for _any_ type of resource, rather than a specific type.
{% endfact %}

Metaparameters follow the familiar `attribute => value` syntax. There are four metaparameter **attributes** that you can include in your resource declaration to order relationships among resources.

* `before` causes a resource to be applied **before** a specified resource
* `require` causes a resource to be applied **after** a specified resource
* `notify` causes a resource to be applied **before** the specified resource, just as with `before`. Additionally, notify will generate a refresh event for the specified resource when the notifying resource changes. 
* `subscribe` causes a resource to be applied **after** the specified resource. The subscribing resource will be refreshed if the target resource changes.

The **value** of the relationship metaparameter is the title or titles (in an array) of one or more target resources. Since this is the first time we've mentioned arrays - here's an example of an array:

{% highlight puppet %}

$my_first_array = ['one', 'two', 'three']

{% endhighlight %}

Here's an example of how the `notify` metaparameter is used:

{% highlight puppet %}
file {'/etc/ntp.conf':
  ensure => file,
  source => 'puppet:///modules/ntp/ntp.conf',
  notify => Service['ntpd'],
}

service {'ntpd':
  ensure => running,
}
{% endhighlight %}

In the above, the file `/etc/ntp.conf` is managed. The contents of the file are sourced from the file `ntp.conf` in the ntp module's files directory. Whenever the file `/etc/ntp.conf` changes, a refresh event is triggered for the service with the title `ntpd`. By virtue of using the notify metaparameter, we ensure that Puppet manages the file first, before it manages the service, which is to say that `notify` implies `before`.

Refresh events, by default, restart a service (such as a server daemon), but you can specify what needs to be done when a refresh event is triggered, using the `refresh` attribute for the `service` resource type, which takes a command as the value.

In order to better understand how to explicitly specify relationships between resources, we're going to use SSH as our example. Setting the `GSSAPIAuthentication` setting for the SSH daemon to `no` will help speed up the login process when one tries to establish an SSH connection to the Learning VM. 

Let's try to disable GSSAPIAuthentication, and in the process, learn about resource relationships.

{% task 1 %}
Create a puppet manifest to manage the `/etc/ssh/sshd_config` file

Create the file `/root/sshd.pp` using a text editor, with the following content in it.

{% highlight puppet %}
file { '/etc/ssh/sshd_config':
  ensure => file,
  mode   => 600,
  source => '/root/examples/sshd_config',
}
{% endhighlight %}

What we have done above is to say that Puppet should ensure that the file `/etc/ssh/sshd_config` exists, and that the contents of the file should be sourced from the file `/root/examples/sshd_config`. The `source` attribute also allows us to use a different URI to specify the file, something we will discuss in the Modules quest. For now, we are using a file in `/root/examples` as the content source for the SSH daemon's configuration file.

Now let us disable GSSAPIAuthentication.

{% task 2 %}
Disable GSSAPIAuthentication for the SSH service

Edit the `/root/examples/sshd_config` file.  
Find the line that reads:

  GSSAPIAuthentication yes

and edit it to read:

  GSSAPIAuthentication no  

Save the file and exit the text editor.

Even though we have edited the source for the configuration file for the SSH daemon, simply changing the content of the configuration file will not disable the GSSAPIAuthentication option. For the option to be disabled, the service (the SSH server daemon) needs to be restarted. That's when the newly specified settings will take effect.

Let's now add a metaparameter that will tell Puppet to manage the `sshd` service and have it `subscribe` to the config file. Add the following Puppet code below your file resource:

{% highlight puppet %}
service { 'sshd':
  ensure     => running,
  enable     => true,
  subscribe  => File['/etc/ssh/sshd_config'],
}
{% endhighlight %}

Notice that in the above the `subscribe` metaparameter has the value `File['/etc/ssh/sshd_config']`. The value indicates that we are talking about a file resource (that Puppet knows about), with the _title_ `/etc/ssh/sshd_config`. That is the file resource we have in the manifest. References to resources always take this form. Ensure that the first letter of the type ('File' in this case) is always capitalized when you refer to a resource in a manifest.

Now, let's apply the change. Remember to check syntax, and do a dry-run using the `--noop` flag first, before using `puppet apply /root/sshd.pp` to apply your changes. 

You will see Puppet report that the content of the `/etc/ssh/sshd_config` file changed. You should also be able to see that the SSH service was restarted. 

In the above example, the `service` resource will be applied **after** the `file` resource. Furthermore, if any other changes are made to the targeted file resource, the service will refresh.

## Package/File/Service

Wait a minute! We are managing the service `sshd`, we are managing its configuration file, but all that would mean nothing if the SSH server package is not installed. So, to round it up, and make our manifest complete with regards to managing the SSH server on the VM, we have to ensure that the appropriate `package` resource is managed as well. 

On CentOS machines, such as the VM we are using, the `openssh-server` package installs the SSH server. 

- The package resource makes sure the software and its config file are installed.
- The file resource config file depends on the package resource.
- The service resources subscribes to changes in the config file.

The **package/file/service** pattern is one of the most useful idioms in Puppet. It’s hard to overstate the importance of this pattern! If you only stopped here and learned this, you could still get a lot of work done using Puppet.

To stay consistent with the package/file/service idiom, let's dive back into the sshd.pp file and add the `openssh-server` package to it.

{% task 3 %}
Manage the package for the SSH server

Type the following code in above your file resource in file `/root/sshd.pp`

{% highlight puppet %}
package { 'openssh-server':
  ensure => present,
  before => File['/etc/ssh/sshd_config'],
}
{% endhighlight %}

- Make sure to check the syntax.  
- Once everything looks good, go ahead and apply the manifest.

Notice that we use `before` to ensure that the package is managed before the configuration file is managed. This makes sense, since if the package weren't installed, the configuration file (and the `/etc/ssh/` directory that contains it would not exist. If you tried to manage the contents of a file in a directory that does not exist, you are destined to fail. By specifying the relationship between the package and the file, we ensure success.

Now we have a manifest that manages the package, configuration file and the service, and we have specified the order in which they should be managed.

## Let's do a Quick Review

In this Quest, we learned how to specify relationships between resources, to provide for better control over the order in which the resources are managed by Puppet. We also learned of the Package-File-Service pattern, which emulates the natural sequence of managing a service on a system. If you were to manually install and configure a service, you would first install the package, then edit the configuration file to set things up appropriately, and finally start or restart the service.


