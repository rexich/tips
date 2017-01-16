# Chef

Chef is a configuration management tool written in Ruby and Erlang. It
uses a pure-Ruby, domain-specific language (DSL) for writing system
configuration "recipes".


## Installation
Chef client:
```sh
$ curl -L https://omnitruck.chef.io/install.sh | sudo bash
```

Chef Development Kit:
```sh
$ curl -L https://omnitruck.chef.io/install.sh | sudo bash -s -- \
-P chefdk -c stable
```


## Usage

- Chef updates what has changed, and makes sure it gets to a consistent
  state defined in the recipes - *test and repair*
- Run a recipe: `chef-client --local-mode hello.rb`
- Another way: `chef-apply recipe.rb`
- `chef-solo` executes `chef-client` in a way that does not require
  Chef server to converge cookbooks, runs in local mode, has no
  centralized distribution of cookbooks or API, no authentication, no
  authorization, can be run as a daemon
  `chef-solo -c ~/chef/solo.rb`
- [Chef-solo](https://docs.chef.io/chef_solo.html) has `solo.rb` as its
  [configuration file](https://docs.chef.io/config_rb_solo.html), where
  you can set paths and recipes to run
- `chef-client` is [an agent](https://docs.chef.io/chef_client.html)
  that runs locally on every node managed by Chef, when it is run it
  will perform all steps required to bring a node to the expected state:
  - registering and authenticating it with the Chef server
  - building the node object
  - synchronizing cookbooks
  - compiling resource collection by loading each cookbook, recipe,
    attribute, all dependencies
  - taking appropriate actions to configure the node
  - looking for exceptions and notifications, handling them as required
- RSA public key-pairs are used to authenticate the chef-client with
  the Chef server every time the client needs to access data stored on
  the server
- Ohai detects attributes on the node and provides them to `chef-client`
  at the start of every client's run
- `knife` is [a tool](https://docs.chef.io/knife.html) that provides an
  interface between local chef-repo and the Chef server, used to manage:
  - nodes, cookbooks, recipes, roles
  - data bags (stores of JSON data, including encrypted ones)
  - environments, cloud resources (including provisioning)
  - installation of `chef-client` on management workstations
  - searching of indexed data on the Chef server


## Recipes

[Recipe](https://docs.chef.io/recipes.html) is a Ruby file that
represents a collection of resources, used to define everything that is
required to configure a part of a system.
It must be stored in a cookbook, may be included in another recipe, may
depend on one or more recipes, must be added to a run list before
running `chef-client`, and is always executed in the same order as is
listed in a runlist.

- Including a recipe: `include_recipe 'recipe'`
- Assign a dependency to a recipe: `depends 'apache2'`


## `file` resource

The [`file` resource](https://docs.chef.io/resource_file.html) allows us
to work with files on the node, including creating, deleting, changing
contents and/or access rights, etc.

```ruby
file 'name' do
  atomic_update              TrueClass, FalseClass # default: true
  backup                     FalseClass, Integer # default: 5
  checksum                   String # default: nothing
  content                    String # what will be written in the file
  force_unlink               TrueClass, FalseClass # default: false
  group                      String, Integer
  inherits                   TrueClass, FalseClass # Windows only
  manage_symlink_source      TrueClass, FalseClass, NilClass # nil
  mode                       String, Integer # ex. 0755, 0666, etc.
  notifies                   # :action, 'resource[name]', :timer
  owner                      String, Integer
  path                       String # default: 'name' if not specified
  provider                   Chef::Provider::File
  rights                     Hash # Windows only
  sensitive                  TrueClass, FalseClass # default: false
  subscribes                 # :action, 'resource[name]', :timer
  verify                     String, Block # perform recipe if given
                                           # code block returns true
  action                     Symbol # default: :create if not specified
end
```
- Actions can be one of: `:create` (default), `:create_if_missing`,
  `:delete`, `:nothing`, `:touch`
- Create a file:
```ruby
file '/tmp/something' do
  owner 'root'
  group 'root'
  mode '0755'
  action :create
end
```
- Delete a file:
```ruby
file '/tmp/something' do
  action :delete
end

```
- Set file modes:
```ruby
file '/tmp/something' do
  mode '0755'
end
```
- Delete a repository and use apt-get to clean cache and update lists:
```ruby
execute 'clean-apt-cache' do
  command 'apt-get clean'
  action :nothing
end

execute 'apt-get-update' do
  command 'apt-get update'
  action :nothing
end

file '/etc/apt/sources.list.d/unnecessary-repo.list' do
  action :delete
  notifies :run, 'execute[clean-apt-cache]', :immediately
  notifies :run, 'execute[apt-get-update]', :immediately
end
```
- Write a string to a file:
```ruby
status_file = '/path/to/file/status_file'

file status_file do
  owner 'root'
  group 'root'
  mode '0755'
  content 'Nyan-cat is omnipresent in space :3'
end
```
- Create a file from a copy:
```ruby
file '/root/lol.txt' do
  content IO.read('/tmp/lol.txt')
  action :create
end
```
