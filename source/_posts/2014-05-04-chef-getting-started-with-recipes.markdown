---
layout: post
title: "Chef - Getting started with recipes"
date: 2014-05-04 17:20:04 -0400
comments: false
categories: [chef, recipe, apache, knife]
---

{% img center /images/recipe_book.jpg %}

Check out my primer chef document for how to initially configure your chef test environment with vagrant. This document assumes you are starting from there.

# Apache Tutorial Cookbook #
These are my expanded notes from one of the official [docs](https://learnchef.opscode.com/tutorials/create-your-first-cookbook/). Before we proceed, let's take a quick look at the default [chef-repo](https://github.com/opscode/chef-repo) tree:

```
chef-repo
├── .chef
├── certificates
├── config
├── cookbooks
├── data_bags
├── environments
└── roles
```

When you are inside that directory and have already configured your knife config file, you can start a cookbook with:

```
knife cookbook create apache_tutorial
```

Let's see how this changes our workspace:

```
chef-repo
└── cookbooks
        └── apache_tutorial
            ├── attributes
            ├── CHANGELOG.md
            ├── definitions
            ├── files
            │   └── default
            ├── libraries
            ├── metadata.rb
            ├── providers
            ├── README.md
            ├── recipes
            │   └── default.rb
            ├── resources
            └── templates
                └── default
```

Make the following modifications:

`recipes/default.rb`:
```
package "apache2" do
  action :install
end

service "apache2" do
  action [ :enable, :start ]
end

cookbook_file "/var/www/index.html" do
  source "index.html"
  mode "0644"
end
```

`files/default/index.html`:
```
<html>
<body>
  <h1>Hello, world!</h1>
</body>
</html>
```

Upload that recipe to the chef server:

```
$ knife cookbook upload apache_tutorial
```

# Run List #

Now we need to add that recipe to our node's `run list`. To verify the run list of a node:

```
$ knife node show chefclient
Node Name:   chefclient
Environment: _default
FQDN:        ChefClient
IP:          10.0.2.15
Run List:    
Roles:       
Recipes:     
Platform:    ubuntu 12.04
Tags:
```

Let's add our apache recipe to that node's runlist and confirm:

```bash
$ knife node run_list add chefclient apache_tutorial
chefclient:
  run_list: recipe[apache_tutorial]

$ knife node show chefclient
Node Name:   chefclient
Environment: _default
FQDN:        ChefClient
IP:          10.0.2.15
Run List:    recipe[apache_tutorial]
Roles:       
Recipes:     
Platform:    ubuntu 12.04
Tags:
```

To force our chefclient to run this recipe and check into our chef server:

```
$ knife ssh "name:chefclient" -x vagrant -P vagrant "sudo chef-client"
```


# References #
* [Chef Resources/Providers Reference](http://docs.opscode.com/chef/resources.html)
* [Chef Tutorial](https://learnchef.opscode.com/tutorials/create-your-first-cookbook/)
* [Chef Query](http://docs.opscode.com/essentials_search.html)
