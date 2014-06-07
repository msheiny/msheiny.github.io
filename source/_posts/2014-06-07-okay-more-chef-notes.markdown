---
layout: post
title: "Chef Recipes Expanded"
date: 2014-05-19 18:19:18 -0400
comments: true
categories: [chef]
---

{% img center /images/chef_on_fire.jpg %}

So here are some chef notes. I watched some videos, read some documentation and 
then I wrote some notes. I should probably find a new hobby.


# The Chef-client run process #

This [page](http://docs.opscode.com/essentials_nodes_chef_run.html) provides a good starting point and diagram. I extracted the high-level steps as follows:

1. *Get configuration data* - Node gets process configuration data from it's own `client.rb` and from `ohai`
2. *authentication* - client authenticates to the chef server using RSA keys and node name. On first run, the chef-validator will be used to generate the RSA private key.
3. *get, rebuild node object* - client pulls down node object from server and rebuilds it.
4. *expand the run-list* - chef client expands the full run-list from the latest node object that should be applied and in what order.
5. *sync cookbooks* - asks the chef for required cookbook files and pulls down any changed and new files (compared to whats in cache from last run)
6. *Reset node attributes* - attributes in rebuilt node object are reset and refilled with attributes from files, environments, roles, and `ohai`.
7. *Compile resource collection* - determine all resources that will be needed to process the run-list
8. *Converge the node* - client configures the system based on info collected
9. *Update the node object, process exception and report handlers* - client updates node object on server and will go through any exception and report handling as necessary.

# Authentication and Authorization #

Communication done via Chef Server API via REST. The server requires authentication via public keys. Public keys stored on chef server, private keys on nodes. The `chef-client` and `knife` tools all use the Chef Server API and will sign a special group of HTTP headers with the private key. 

During the first `chef-client` run, there is no private key so the chef-client will try the private key assigned to the chef-validator. Assuming that goes successfully, the chef-client will obtain a `client.pem` private key for future authentication. 

The chef-client authentication workflow:

1. Look in `/etc/chef/client.pem` for a node private key
2. If that doesn't work, check the validator in `/etc/chef/validation.pem`

During an initial knife bootstrap, knife will copy the validation pem over to the node to do an initial chef-client run.

# Compile and Execute #
This stage iterating through our run list resources, collecting that into a list, and then iterating over that on the client to determine next action. Chef client will compare to see if the resource is in the appropriate state. If it is already there, no action is taken. Otherwise, the client will work to put it in the desired state.

# Node Object #
* Any machine configured to be maintained by Chef
* Node object is always available to you from within a recipe
* Each node must have a unique name within an organization. Chef defaults to the FQDN.

* Nodes are made up of attributes
    - Many are discovered automatically (`ohai`)
    - Other objects in Chef can add Node attributes (cookbooks, roles/environments, recipes, attribute files)
    - Nodes are stored and indexed on the Chef server

You can quickly get various attribute data from your node through `knife` or `ohai`

```bash
# From the chef node directly
$ sudo ohai 

# Or from the knife output
$ knife node list chefclient -l

# You can modify formatting options 
# $ knife node list {clientname} -F{format}
$ knife node list chefclient -Fj 
{
  "name": "chefclient",
  "chef_environment": "_default",
  "run_list": [
    "recipe[apt]",
    "recipe[apache_tutorial]"
  ],
  "normal": {
    "tags": [
    ]
  
  }
}

# List a specific attribute with -a
# $ knife node list clientname -a {attribute}
$ knife node list chefclient -a platform_version
chefclient:
  platform_version: 12.04
```

Knife searches are based on Apache Solr and are in the format:

```
key:search_pattern
```

* Key = field name found 
* search pattern = what will be searched for

Examples:

```bash
# Wildcard over everything
knife search node '*:*'

# Search for node running ubuntu
knife search node 'platform:ubuntu'
```

# Templates and Cross platform #
You can utilize Ruby within the recipe which can help to create dynamic run-lists. Combining variables, loops, case statements, etc. can make for some powerful conditional recipes. 

Variable usage:

```ruby

package_name = "apache2"

if node[:platform] == "centos"
    package_name = "httpd"
end

package package_name do
  action :install
end
```

Case usage:

```ruby
package "apache2" do
  case node[:platform]
  when "centos","redhat","fedora","suse"
    package_name "httpd"
  when "debian","ubuntu"
    package_name "apache2"
  when "arch"
    package_name "apache"
  end
  action :install
end
```

Array iteration:

```ruby
["apache2", "apache2-mpm"].each do |p|
  package p
end
```

We can utilize an Embedded Ruby template instead of static files that will be dynamically modified when copied down to our node. These are commonly named with a `.erb` suffix and can contain ruby code within text. In order to embed ruby we will need to utilize the `<%= %>` tag:

```html
<html>
<p>
    Hello, <%= "my name is #{$ruby}" %>
</p>
</html>
```

We can feed in a template file with the `template` resource like this:

```
template "/etc/sudoers" do
  source "sudoers.erb"
  mode 0440
  owner "root"
  group "root"
  variables({
     :sudoers_groups => node[:authorization][:sudo][:groups],
     :sudoers_users => node[:authorization][:sudo][:users]
  })
end
```

The sudoers.erb file referenced above will need to go into the `templates/default/sudoers.erb` instead of a typical file path at `files/default/sudoers.erb`. Chef will try to match up the most specific file by the client platform though and will look in this order:

`templates/`
1. `host-node[:fqdn]/`
2. `node[:platform]-node[:platform_version]/`
3. `node[:platform]-version_components/`
4. `node[:platform]/`
5. `default/`

For example, you could store the file in `templates/ubuntu-12.04/sudoers.erb`.

# Attributes #

`Attributes` are specific details about a node and can be assigned from one of the following locations:

* Nodes (collected by Ohai at the start of each chef-client run)
* Attribute files (in cookbooks)
* Recipes (in cookbooks)
* Environments
* Roles

The precedence level that determines which attribute is applied is as follows :

1. A default attribute located in a cookbook attribute file
2. A default attribute located in a recipe
3. A default attribute located in an environment
4. A default attribute located in role
5. A force_default attribute located in a cookbook attribute file
6. A force_default attribute located in a recipe
7. A normal attribute located in a cookbook attribute file
8. A normal attribute located in a recipe
9. An override attribute located in a cookbook attribute file
10. An override attribute located in a recipe
11. An override attribute located in a role
12. An override attribute located in an environment
13. A force_override attribute located in a cookbook attribute file
14. A force_override attribute located in a recipe
15. An automatic attribute identified by Ohai at the start of the chef-client run

You can more directly influence precedence by taking advantage of attribute types when getting and setting. The types available are (in order of precedence, lowest to highest):

* `default` - lowest precedence, should be used in cookbooks when possible
* `force_default` - force attribute in cookbook to take over default set in an environment  or role
* `normal` - not reset during chef-client run and has higher precedence than default
* `override` - reset at start of each client run
* `force_override` - ensure override in cookbook takes over for an override attribute in a role or environment
* `automatic` - discovered by `ohai` and has highest precedence


To store attributes in a cookbook, we can place them in `attributes/default.rb`:

```
# node object is implicit so we can actually do either of the following:
node.default[:apache][:dir]          = "/etc/apache2"
node.default[:apache][:listen_ports] = [ "80","443" ]

# OR 
default[:apache][:dir]          = "/etc/apache2"
default[:apache][:listen_ports] = [ "80","443" ]
```


# Roles #
A good way to bridge together cookbooks and attributes for re-use. To start a role, start by creating an `.rb` file in your `roles/` folder in the chef repository. Here is an example of the three required attributes:

```ruby
# roles/webserver.rb
name "webserver"
description "The base role for systems that serve HTTP traffic"
run_list "recipe[apache2]", "recipe[apache2::mod_ssl]", "role[monitor]"
```

Common role tasks in knife:

```bash
# Upload a role
knife role from file webserver.rb

# Show roles
knife role list

# Show details on a specific role
knife role show webserver

# Delete a role
knife role delete webserver

# Add a role to a node
knife node run_list add chefclient 'role[webserver]'
```

# Environments #

A way to logically break up workflow and reflect organizational patterns. 
Every node can be assigned to a single environment. By default, all nodes 
are part of the `_default` environment. You can create a new environment by creating a *.rb file in the `environments` folder of the chef-repo. This process is very similar to creating a new role. Your environment file needs 
a `name`, `cookbook`, `description` (`default_attributes` and `override_attributes` are optional):

```ruby
# environments/dev.rb
name "dev"
description "The development environment"
cookbook_versions  "couchdb" => "= 11.0.0"
default_attributes "apache2" => { "listen_ports" => [ "80", "443" ] }
```

Here are some environment management commands from knife:

```bash
# Create environment from dev.rb file
knife environment create from file dev.rb

# Show available environments
knife environment list

# Show environment assignment for a node
knife node show $nodename | grep ^Env
```

It appears you'll have to use a plugin to knife to modify environments but otherwise it is a very trivial task to perform from the chef server web interface. 

# Community Cookbooks #

There is a great online resource for interacting with cookbooks from others online at <http://community.opscode.com/cookbooks>. Once you find a good looking cookbook you'd like to pull into your own environment (after proper vetting of course), you can use knife to assist here:

```bash

# search the community site
knife cookbook site search chef-client

# show detail about a particular package
knife cookbook site show chef-client

# download community cookbook into local tar file
knife cookbook site download chef-client 

# Install cookbook into cookbooks directory and commit changes
knife cookbook site install chef-client

```

From what I can tell, using the `site install` command is against chef best practices. You really should be vetting third-party code before pulling it down and committing it to your environment. There is a great community of chef tools and knife plugins out there. The Opscode folks have singled out some of the most common in the [Chef Development Kit](http://www.getchef.com/downloads/chef-dk/).

Okay! Great notes everyone! 

# Reference #

* [Fundamentals Webinar - Module 4](http://youtu.be/H3Ggublb378)
* [Fundamentals Webinar - Module 5](http://youtu.be/46NQNzkCe_U)
* [Chef Run](http://docs.opscode.com/essentials_nodes_chef_run.html)
* [Authentication and Authorization](http://docs.opscode.com/auth.html)
* [Searching](http://docs.opscode.com/essentials_search.html)
* [Recipe Docs](http://docs.opscode.com/essentials_cookbook_recipes.html)
* [Attributes](http://docs.opscode.com/chef_overview_attributes.html)
