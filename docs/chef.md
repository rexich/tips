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

A [recipe](https://docs.chef.io/recipes.html) is a Ruby file that
represents a collection of resources, used to define everything that is
required to configure a part of a system. In other words, it is an
ordered series of configuration states.
It must be stored in a cookbook, may be included in another recipe, may
depend on one or more recipes, must be added to a run list before
running `chef-client`, and is always executed in the same order as is
listed in a runlist.

- Including a recipe: `include_recipe 'recipe'`
- Assign a dependency to a recipe: `depends 'apache2'`


## Resources

A [resource](https://docs.chef.io/resource.html) is a policy statement
that describes the desired state of a part of the system.
It also describes the steps to bring that part to the desired state.

Several resources are grouped into recipes. Resources declare what state
part of the system should be in, but not how to get to that state - that
is handled by Chef, so we don't have to worry about it.

Resources can generate a file, install a package, configure a service.
All resources have actions, and if an action is not given, usually the
default (more permissive) action is performed.


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

- Action can be one of: `:create` (default), `:create_if_missing`,
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


## `apt_update` resource

The [`apt_update` resource](https://docs.chef.io/resource_apt_update.html)
can be used to manage Apt repository updates of software lists on Ubuntu
and Debian nodes.

```ruby
apt_update 'name' do
  frequency                  Integer  # number of seconds to wait until
                                      # next update to be performed
  action                     Symbol # default: :periodic if unspecified
end
```

- Action can be one of: `:nothing`, `:periodic` (default), or `:update`
- `:periodic` will update each time the `frequency` number of seconds
  passes, and `:update` will update at each run of `chef-client`
- Example: update the cache every 24 hours (86400 seconds)
```ruby
apt_update 'update apt cache every day' do
  frequency 86_400
  action :periodic
end
```


## `group` resource

The [`group` resource](https://docs.chef.io/resource_group.html) lets us
manage groups on a node.

```ruby
group 'name' do
  append                     TrueClass, FalseClass # default: false
  excluded_members           Array # ['user1, 'user2']
  gid                        String, Integer  # group ID
  group_name                 String # default: 'name' if not specified
  members                    Array  # ['user1, 'user2']
  non_unique                 TrueClass, FalseClass  # avoids group ID
                                # duplication, default: false
  notifies                   # see description
  provider                   Chef::Provider::Group
  subscribes                 # see description
  system                     TrueClass, FalseClass  # show if a group
                                # belongs to a system group
  action                     Symbol # default: :create if not specified
  ignore_failure             TrueClass, FalseClass # default: false
end
```

- Action can be one of: `:create` (default), `:manage` (does not rise an
  exception if group does not exist, `:modify` (for existing group only,
  rises an exception if group does not exist), `:nothing`, `:remove`
- If `append true`, adds `members` to the `group_name` or `name`, and
  removes `excluded_members` from the `group_name` or `name`
- Append users to a group:
```ruby
group 'www-data' do
  action :modify
  members ['maintenance', 'writers']
  append true
end
```
