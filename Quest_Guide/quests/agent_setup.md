---
title: Agent Node Setup
layout: default
---

# Agent Node Setup

## Quest objectives

- Learn how to install the Puppet agent on a node.
- Use the PE console to sign the certificate of a new node.
- Understand a simple Puppet architecture with a Puppet master
  serving multiple agent nodes.
- Use the `site.pp` manifest to classify nodes.

## Getting Started

So far, you've been managing the Puppet master server using the Puppet agent on
that node. Though you certainly can use Puppet to configure your Puppet master
server, most of the work you'll be doing with Puppet will be on separate nodes.

In this quest, we'll use a tool called `docker` to simulate multiple nodes on
the Learning VM. With these new nodes, you can learn how to install the puppet
agent, sign the certificates of your new nodes to allow them to join your
puppetized infrastructure, and finally use the `site.pp` manifest to apply some
simple Puppet code on these new nodes.

When you're ready to get started, type the following command:

    quest --start agent_node_setup

## agent node setup

> A quote

> -The person who said it

So far, we've been using two different Puppet commands to apply our Puppet code:
`puppet apply`, and `puppet agent -t`. If you haven't felt confident about the
distinction between these two commands, it could be because we've been doing
everything on a single node where the difference between applying changes
locally and involving the Puppet master isn't clear. Let's take a moment to 
review.

`puppet apply` compiles a catalog based on a specified manifest and applies that
catalog locally. Any node with the Puppet agent installed can run a `puppet apply`
locally. You can get quite a bit of use from `puppet apply` if you want to use
Puppet on an agent without involving a Puppet master server. For example, if you
are doing local testing of Puppet code or experimenting with a small infrastructure
without a master server.

`puppet agent -t` triggers a Puppet run. This Puppet run is a conversation between
the agent node and the Puppet master. First, the agent sends a collection of facts
to the Puppet master. The master takes these facts and uses them to determine what
Puppet code should be applied to the node. You've seen two ways that this
classification can be configured: the `site.pp` manifest and the PE console node
classifier. The master then evaluates the Puppet code to compile a catalog that
describes exactly how the resources on the node should be configured. The master
sends that catalog to the agent on the node, which applies it. Finally, the agent
sends its report of the Puppet run back to the master. By default, these Puppet
runs are scheduled to happen automatically every half hour. We have disable
these automatic runs on the Learning VM, to provide you explicit control over
when Puppet runs (and reconfigures the node as needed). 

Though you only need a single node to learn to write and apply Puppet code, getting
the picture of how the Puppet agent and master nodes communicate will be much
easier if you actually have more than one node to work with.

### docker docker docker

We've created a `multi_node` module that will set up a pair of docker containers
to act as additional agent nodes in your infrastructure. Note that docker is not
a part of puppet; it's an open-source tool we're using to help build a multi-node
learning environment. While running a Puppet agent on a docker container on a
VM gives us a convenient way to see how Puppet works on multiple nodes, it isn't a
recommended way to set up real-life Puppet infrastructure! Unless, of course,
your infrastructure consists primarily of docker containers.

{% task 1 %}
---
- execute:
    - vim /etc/puppetlabs/code/environments/production/manifests/site.pp
    - /node default {\rOnode learning.puppetlabs.vm {\r  include multi_node\r}
    - :wq
{% endtask %}

To apply the `multi_node` class to the Learning VM, add it to the
`learning.puppetlabs.vm` node declaration in your master's `site.pp` manifest.

    vim /etc/puppetlabs/code/environments/production/manifests/site.pp

Insert `include multi_node` into the `learning.puppetlabs.vm` node declaration.

{% highlight Puppet %}
node learning.puppetlabs.vm {
  include multi_node
}
{% endhighlight %}

(Note that it's important that you don't put this in your `default` node
declaration. If you did, Puppet would try to create docker containers on your
docker containers every time you did a Puppet run!)

{% task 2 %}
---
- execute: puppet agent -t
{% endtask %}

Now trigger an agent run to apply the class. Note that this might take a little
while to run.

  puppet agent -t

Once this run has completed, you can use the `docker ps` command to see your two
new nodes. You should see one called `database` and one called `webserver`.

### install the Puppet agent

Now you have two fresh nodes, but you don't have the Puppet agent installed on
either! Installing the agent will be the first step of getting these nodes into
our Puppet infrastructure.

{% task 3 %}
---
- execute: puppet agent -t
{% endtask %}

Your Puppet master hosts an agent installation script, so in most cases the
simplest way to install an agent is to use the `curl` command to grab this
script from the master and execute it.

For easy reference, you can find this `curl` command on the PE console.
Navigate to `https://<VM's IP address>` in your browser address bar.

Remember, use the following credentials to connect to the console:

* username: **admin**
* password: **puppetlabs**

Navigate to the *Node Management* > *Inventory* and look under the
*Unsigned certificates* tab. You will see a section with the heading
**Adding nodes to manage wtih Puppet Enterprise**. Beneath this, you will
find a curl script that you can copy to any nodes you wish to manage with
puppet.

  curl -k https://learning.puppetlabs.vm:8140/packages/current/install.bash | sudo bash

Ordinarily, you would probably use `ssh` to connect to your agent nodes and
run this command. Because we're using docker, however, the way we connect will
be a little different. To connect to your `webserver` node, run the following
command to execute an interactive bash session on the container.

  docker exec -it webserver bash

Then paste in the `curl` command from the PE console to install the Puppet agent
on the node. When the installation completes, end your bash process on the
container:

  exit

And repeat the process with your database node:

  docker exec -it database bash

Great, now you have two new nodes with the Puppet agent installed! With the agent
installed, you will have access to all of the Puppet commands and related tools
we've covered so far, including `puppet resource`, `puppet agent`, `puppet apply`
and `facter`.

{% task 4 %}
---
- execute: puppet agent -t
{% endtask %}

We will have to sign each new node's certificate on the master before we will be
able to trigger a Puppet run with `puppet agent -t`. We will get to that in a moment
but first, let's take a moment to see what we can do with just the agent installed
and no interaction with the Puppet master.

Let's use facter to get some information about this node:

  facter operatingsystem

You can see that though the Learning VM itself is running CentOS, our new nodes
run Ubuntu.

  facter certname

We can also see that this node's certname is `database.learning.puppetlabs.vm`. This
is how we can identify the node in the PE console or the `site.pp` manifest on
our master.

{% task 5 %}
---
- execute: puppet agent -t
{% endtask %}

Next, let's try some basic Puppet commands. We can use the Puppet resource tool
to easily create a new test file.

  Puppet resource file /tmp/test ensure=file

You'll see your new file created.

  Notice: /File[/tmp/test]/ensure: created
  file { '/tmp/test':
    ensure => 'file',
  }

{% task 6 %}
---
- execute: puppet agent -t
{% endtask %}

You can also use the `puppet apply` command to apply the contents of a manifest.
Create a simple test manifest to give it a try.

  vim /tmp/test.pp

We'll define a simple message:

{% highlight Puppet %}
  notify { "Hi, I'm a manifest applied locally on an agent node": }
{% endhighlight %}

And apply it:

  puppet apply /tmp/test.pp

You should see the following output:

  Notice: Compiled catalog for database.learning.puppetlabs.vm in environment production in 0.32 seconds
  Notice: Hi, I'm a manifest applied locally on an agent node!
  Notice: /Stage[main]/Main/Notify[Hi, I'm a manifest applied locally on an agent node!]/message: defined 'message' as 'Hi, I'm a manifest applied locally on an agent node!'
  Notice: Applied catalog in 0.02 seconds

{% task 4 %}
---
- execute: puppet agent -t
{% endtask %}

So that's what you can do on an agent node, but what are the differences?
First, let's take a look at the directory where you'd find all your puppet
code on the master. Check:

  ls /etc/puppetlabs/code/environments/production/manifests

and

  ls /etc/puppetlabs/code/environments/production/modules

There's nothing there—no modules or `site.pp` manifest! This is because
all the code in your Puppet infrastructure should live on your puppet
master, not your agent nodes. When a Puppet run is triggered—either
as scheduled or manually with the `puppet agent -t` command, it is the
puppet master that compiles your Puppet code into a manifest to be applied
on your agent node.

So let's give it a try. Go ahead and trigger a Puppet run on your `database`
node.

  puppet agent -t

You'll see that instead of completing a Puppet run, Puppet exits with the
following message:

  Exiting; no certificate found and waitforcert is disabled

### Certificates

Though you've created your new nodes and installed the Puppet agent, there's
one more step before they can join your Puppet infrastructure. The Puppet master
keeps a list of signed certificates for each node in your infrastructure. This
helps keep your infrastructure secure and prevents Puppet from making unintended changes
to systems on your network.

{% task 4 %}
---
- execute: puppet agent -t
{% endtask %}

Before we can run Puppet on our new agent nodes, we'll need to sign their certificates
on the Puppet master. If you're still connected to your agent node, return to the master:

  exit

Use the `puppet cert list` command to list the unsigned certificates. (You could
also view and sign these from the inventory page of the PE console.)

  puppet cert list

Now sign each of your nodes' certificates:

  puppet cert sign webserver.learning.puppetlabs.vm

and

  puppet cert sign database.learning.puppetlabs.vm

Now your certificates are signed, so your new nodes will be managed by puppet.
To test this out, let's add a simple `notify` resource to the `site.pp` manifest
on the master.

  vim /etc/puppetlabs/code/environments/production/manifests/site.pp

Find the `default` node declaration, and edit it to add a `notify` resource
that will tell us some basic information about the node.

{% highlight Puppet %}
node default {
  notify { "This is ${::fqdn}, running the ${::operatingsystem} operating system": }
} 
{% endhighlight %}

Now let's connect to our database node again.

  docker exec -it database bash

And try another Puppet run:

  puppet agent -t

## Review

Review
