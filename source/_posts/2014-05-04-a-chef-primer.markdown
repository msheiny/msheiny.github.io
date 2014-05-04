---
layout: post
title: "a Chef Primer"
date: 2014-05-04 17:19:40 -0400
comments: false
categories: [chef, vagrant]
---
## Chef Primer ##

### Vagrant ###
My focus was to learn about Chef while using Vagrant for easy machine management. I'm pretty familiar with the basics of vagrant already but there were two extras I needed for this project:
1. Quick snapshoting capabilities
2. Multi-machine setup for a Chef server and node


For 1. turns out you need a plugin for that. There are a few out there but I went with
[vagrant-vbox-snapshot](https://github.com/dergachev/vagrant-vbox-snapshot).

Snapshot plugin installation:
```
vagrant plugin install vagrant-vbox-snapshot
```

The base commands I care about are:
```
vagrant snapshot take <machine_name> $snapshotname
vagrant snapshot back <machine_name>
```

[Multi-machine](https://docs.vagrantup.com/v2/multi-machine/) mode was fairly straightforward to configure. Here are the relevant pieces from my `Vagrantfile`:

```
Vagrant.configure("2") do |config|

    config.vm.define "chef-server", primary: true do |chefserver|
        chefserver.vm.network :private_network, ip: "192.168.50.100"
        chefserver.vm.box = "precise64"
    end

    config.vm.define "chef-node" do |chefnode|
        chefnode.vm.network :private_network, ip: "192.168.50.10"
        chefnode.vm.box = "precise64"
    end

    end

```

To jump between these two boxes you'll now need to specify the machine name. I 
configured the chef server as the default machine with `primary: true` but you'll need
to refer to `chef-node` by name in your vagrant commands. For example, you can do
`vagrant ssh chef-node`.


### Chef Notes ###

Chef key terms:

- `server` - stores infrastructure pieces
- `workstation` - where you interact with chef server
- `nodes` - a node in your infrastructure
- `resources` - items to be manipulated (directories, files, etc.)
    + Represents a piece of the system and state. Such as a package to be installed, service to be running, file to be generated, user to be managed, etc.
- `recipe` - configuration files that describe resources and their desired state. the "work-horse" of chef
- `cookbook` - hold recipes, templates, files, and custom resources.
- `organizations` - enterprise chef independent tenants
- `environments` - start with a single environment, can tag different life-stages and contain different data attributes.
- `roles` - represent types of servers. roles define policies. 
- `run list` - list of chef configuration files that should be applied
- `nodes` -  represent the machines to be managed. can have 0+ roles. belong to one environment and one organization

`chef-client`
* Runs on each node
* gathers current system config and
* pulls down policy from chef server and enforce the policy on node

#### High-level execution steps ####

How Chef brings a machine in-line with policy:
1. Chef client periodically polls chef server to determine what policies should be running
2. The chef server figures that out and provides the client with a `run list`
3. The chef-client looks at each recipe in the `run list` and determines if it is in line with that policy.

### Chef-server Install ###
1. Download system package from chef install site
2. Depending on your platform, you'll probably utilize 'rpm' or 'dpkg' to install that
3. Ensure that DNS is properly configured for the server, or fake it by editing the `/etc/hosts` file
3. run `sudo chef-server-ctl reconfigure` which kicks off what essentially looks like a chef cookbook to build the chef server
4. You'll now have access to a chef web front-end on the server on standard web ports
5. You can login with admin / p@ssw0rd1

### Chef workstation Install ###
This is the computer(s) where you will manage the chef cookbooks and recipes. You should always be working from within your chef-repo (see further down).


#### Quick install method ####

```bash
curl -L https://www.opscode.com/chef/install.sh | sudo bash
```

This includes:
* ruby
* knife
* chef-client
* ohai

You can also utilize ruby's gem installer with:

```bash
sudo gem install chef 
#or 
gem install --user-install chef
export PATH="~/.gem/ruby/1.9.1/bin/:$PATH"
```

#### Chef-repo ####

To set-up a propery formatted chef repo, start from https://github.com/opscode/chef-repo.git. You'll need to pull down a template repository and then link your repository to our chef server.

```bash
git clone https://github.com/opscode/chef-repo.git
mkdir chef-repo/.chef
touch chef-repo/.chef/chef-validator.pem
touch chef-repo/.chef/admin.pem
```

Next, you'll need to pull in the certificates from the server. Log back into the web interface. Click `Clients` tab, then Chef-validator's `edit`, and regenerate the `Private key`. Copy the private key text from that page into the `chef-validator.pem` file we just created earlier. Do the same thing for the admin users, start with the `Users` tab and then dump the admin user's private key into the `admin.pem` file we created earlier.

#### Knife ####
knife is one of the primary command-line tools for managing chef instances. In order to bootstrap our knife configuration, you'll need to run the knife initial command:

```
[msheiny:~/git/chef-repo] ± knife configure -i 
WARNING: No knife configuration file found
Where should I put the config file? [/home/msheiny/.chef/knife.rb] .chef/knife.rb
Please enter the chef server URL: [https://sheiny-desktop.sheiny.com:443] https://chefserver                                                 
Please enter a name for the new user: [msheiny] 
Please enter the existing admin name: [admin] 
Please enter the location of the existing admin's private key: [/etc/chef-server/admin.pem] .chef/admin.pem
Please enter the validation clientname: [chef-validator]      
Please enter the location of the validation key: [/etc/chef-server/chef-validator.pem] .chef/chef-validator.pem
Please enter the path to a chef repository (or leave blank): /home/msheiny/git/chef-repo 
Creating initial API user...
Please enter a password for the new user: 
Created user[msheiny]
Configuration file written to /home/msheiny/git/chef-repo/.chef/knife.rb
```

You can commit the repository at this time if you want. Just be weary of ensuring that the *pem files do not get commited or you might want to exclude the entire `.chef` directory altogether.

Test to make sure everything is working correctly by running - `knife user list`

#### Chef client Install ####
We can bootstrap a chef client into our instance with help from knife!

```bash
[msheiny:~/git/chef-repo] ± knife bootstrap chefclient.sheiny.com -x vagrant -P vagrant -N chefclient --sudo
```
The third argument is the fqdn of the node, `-x` and `-P` are the username/password, and `-N` is the name we want to refer to our new client by.

After that is done we can confirm with `client list` and we should see `chefclient` as part of the list:
```bash
[msheiny:~/git/chef-repo] ± knife client list
```


## Resources ##
* [Chef Install Site](http://www.getchef.com/chef/install/)
* [Third-party Chef Install Tutorial](https://www.digitalocean.com/community/articles/how-to-install-a-chef-server-workstation-and-client-on-ubuntu-vps-instances)
