# Chef 101: Learning Server Provision with Chef

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


## Nodes and Attributes

A [node](https://docs.chef.io/nodes.html) represents a machine managed
by Chef, such as: server, cloud instance, virtual machine, network
device, or virtualized container.

Each node is bootstrapped the first time and will have `chef-client`
installed and the Chef server certificate provided and stored for safe
communication and authentication

Each node's specific details are gathered by Ohai at each `chef-client`
run and provided as [attributes](https://docs.chef.io/attributes.html),
such as an IP address, kernel version, list of loaded kernel modules,
and so on. They can also be set and modified from the defaults in a
cookbook, recipes, roles, and environments. Attributes lists are built
from Ohai data, the node object stored on Chef server, and the updated
generated node object from the current `chef-client` run.

Attributes can be defined in the `attributes/` subfolder of a cookbook,
the default one being `default.rb`. Writing `node` is unnecessary since
it is implicitly referred to.

```ruby
node.default['apache']['dir']          = '/etc/apache2'
node.default['apache']['listen_ports'] = [ '80','443' ]
```

Most frequently used attributes include:

- `node['platform']` - platform on which a node is running, helps\
  determine which providers will be used
- `node['platform_version']` - version of the platform
- `node['ipaddress']` - IP address for a node; if the node has a default
  route, this is the IPv4 address for the interface, otherwise the value
  should be nil; recommended value: the IP address for default route
- `node['macaddress']` - MAC address for a node, determined by the same
  interface that detects the `node['ipaddress']`
- `node['fqdn']` - fully qualified domain name for a node, used as the
  name of a node unless otherwise set
- `node['hostname']` - host name for the node
- `node['domain']` - domain for the node
- `node['recipes']` - a list of recipes associated with a node (and part
  of that node's run-list)
- `node['roles']`  - a list of roles associated with a node (and part of
  that node's run-list)
- `node['ohai_time']` - time at which Ohai was last run; not commonly
  used in recipes, but it is saved to the Chef server and can be
  accessed using the `knife status` subcommand

Use method `attribute?('attr_name')` to check whether attribute is
defined and exists, useful for `if` conditionals.

Nodes can be managed using several available utilities:

- `knife` can be used to create, edit, view, list, tag, and delete nodes
- knife plug-ins can be used to create, edit, and manage nodes that are
  located on cloud providers
- `chef-client` can be used to manage node data using the command line
  and JSON files; each JSON file contains a hash, the elements of which
  are added as node attributes; also, the run_list setting allows roles
  and/or recipes to be added to the node
- `chef-solo` can be used to manage node data using the command line and
  JSON files; each JSON file contains a hash, the elements of which are
  added as node attributes; also, the run_list setting allows roles
  and/or recipes to be added to the node
- The command line can also be used to edit JSON files and files that
  are related to third-party services, such as Amazon EC2, where the
  JSON files can contain per-instance metadata that is stored in a file
 on-disk and then read by chef-solo or chef-client as required


## `chef-client` and `chef-solo`

The `chef-client` is [an agent](https://docs.chef.io/chef_client.html)
that runs locally on every node managed by Chef server, and once it is
run it will perform all steps required to bring a node to the expected
state, and will skip configuring what is already in the wanted state
(*test and repair* method). It handles the following:

- registering and authenticating a node with the Chef server
- building the node object
- synchronizing cookbooks
- compiling resource collection by loading each cookbook, recipe,
  attribute, and all dependencies
- taking appropriate actions to configure the node
- looking for exceptions and notifications, handling them as required

RSA public key-pairs are used to authenticate the chef-client with Chef
server every time the client needs to access data stored on the server.

Ohai detects attributes on the node and provides them to `chef-client`
at the start of every client's run.

The [`chef-solo`](https://docs.chef.io/chef_solo.html) tool can be used
to run `chef-client` in a way that does not require a Chef server to
converge cookbooks, in a so-called 'local mode'.

- `chef-solo` has no centralized distribution of cookbooks or API
- It lacks authentication and authorizatio
- Can be run as a daemon
- Use its [configuration file](https://docs.chef.io/config_rb_solo.html)
  `solo.rb` to set paths and recipes to run
- Usage: `chef-solo -c ~/chef/solo.rb`

You can also use `chef-client` in 'local mode' to run individual recipes
without contacting the Chef server: `chef-client --local-mode hello.rb`;
or you simply run `chef-apply recipe.rb`, but it does not support
templates at all.


## `knife` management tool
The [Knife](https://docs.chef.io/knife.html) utility provides an
interface between local chef-repo and the Chef server. It is frequently
used to manage the following:

- nodes, cookbooks, recipes, roles
- data bags (stores of JSON data, including encrypted ones)
- environments, cloud resources (including provisioning)
- installation of `chef-client` on management workstations
- searching of indexed data on the Chef server

Knife is a versatile tool with plenty of uses:

- Bootstrapping a node means installing `chef-client` on it and checking
  it in with the Chef server for the first time, and that is performed
  using `knife` over a SSH connection:
```sh
$ knife bootstrap localhost --ssh-port 2222 --ssh-user vagrant --sudo --identity-file chef-server/.vagrant/machines/node1-ubuntu/virtualbox/private_key --node-name node1-ubuntu --run-list 'recipe[learn_chef_apache2]'
```
- List all nodes associated with the Chef server:
```sh
$ knife node list
```
- View information about a node:
```sh
$ knife node show node_name
```
- On each change of functionality (recipes) of your cookbook, you need
 to increase the version number in the `metadata.rb` file and upload
 the cookbook again on the Chef server:
```sh
$ knife cookbook upload le_cookbooke
```
- Then, run Chef client on the node again to apply new configuration:
```sh
$ knife ssh localhost --ssh-port 2222 'sudo chef-client' --manual-list --ssh-user vagrant --identity-file chef-server/.vagrant/machines/node1-ubuntu/virtualbox/private_key
```

In practice, `chef-client` is set up to run periodically and get new
configuration data from the Chef server; refer to the example in Roles.

Deleting data from the Chef server is performed like this:

- Delete node data from Chef server:
```sh
$ knife node delete node1-ubuntu --yes
```
- Delete node client connection from Chef server:
```sh
$ knife client delete node1-ubuntu --yes
```
- Delete cookbook from Chef server:
```sh
$ knife cookbook delete learn_chef_apache2 --all --yes
```
- Delete role from Chef server:
```sh
$ knife role delete web --yes
```
- Delete RSA private key from your node - log into the node first:
```sh
$ sudo rm /etc/chef/client.pem
```


## `berks` cookbook manager

[Berkshelf](https://docs.chef.io/berkshelf.html) is a dependency manager
for Chef cookbooks. It lets the user specify the cookbooks their project
needs, and will automatically resolve which cookbooks depend on the
specified ones and download them all. Afterwards, they can be uploaded
in bulk to the Chef server, instead of one-by-one using `knife`.

Getting cookbooks and their dependencies from the supermarket is done
by writing a Berksfile in the project's top directory and provide a
source URL and the names of the cookbooks to be installed:

```ruby
source 'https://supermarket.chef.io'
cookbook 'chef-client'
```

Afterwards, tell Berks to fetch the cookbooks and their dependencies
and store them locally on your workstation:

```sh
$ berks install
```

Then we use Berks to upload all of the cookbooks to our Chef server,
instead of using Knife to upload them one-by-one:

```sh
$ SSL_CERT_FILE='.chef/trusted_certs/chef-server_test.crt' berks upload
```

You must provide the `SSL_CERT_FILE` environmental variable with the
path to the Chef server's SSL certificate, so the connection can be
eastablished successfully.


## Recipes

A [recipe](https://docs.chef.io/recipes.html) is a Ruby file that
represents a collection of resources, used to define everything that is
required to configure a part of a system. In other words, it is an
ordered series of configuration states, and is the most fundamental
configuration element within an organization.

It must be stored in a cookbook, may be included in another recipe, may
depend on one or more recipes, must be added to a run list before
running `chef-client`, and is always executed in the same order as is
listed in a runlist. You can add helper code in it, since it is a Ruby
program.

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

Resources have a name, a type, parameter attributes, and an action.


## Cookbooks

A [cookbook](https://docs.chef.io/cookbooks.html) is the fundamental
unit of configuration and policy distribution. In essence, it defines
a scenario and contains all things required to support the screnario,
such as: recipes, attribute values, templates, extensions (libraries).

Cookbooks help us stay organized by adding structure to our work and by
grouping recipes. Default cookbook settings are kept in `default.rb` and
are loaded first, and then other configuration files are loaded in
alphabetical order, and can override settings in this file.


- Generate a cookbook:
```sh
$ mkdir cookbooks
$ chef generate cookbook cookbooks/apache2
```
- Generate a template:
```sh
$ chef generate template cookbooks/apache2 index.html
```
- Run a cookbook locally on the running node:
```sh
$ sudo chef-client --local-mode --runlist 'recipe[learn_chef_apache2]'
$ sudo chef-client --local-mode --runlist 'recipe[lol::default]'
```
- Upload cookbook to Chef server and list them on server:
```sh
$ knife cookbook upload le_cookbooke
$ knife cookbook list
```


## Roles

[Roles](https://docs.chef.io/roles.html) are a way to define certain
patterns and processes that exist across nodes in an organization and
that belong to a single job function, like: web server, backup server,
file server, etc.

Each role contains zero or more attributes and a run-list, and each node
can have zero or more roles assigned to it. The attributes a role has
can either be `default` or `override`, and the role's attributes are
compared to the node's ones, and change the node's attributes if needed.

At the beginning of a `chef-client` run, all attributes are reset, and
rebuilds them using automatic attributes provided by Ohai at the start
of the run, and then it uses `default` and `override` attributes
specified in cookbooks or by roles and environments.

Normal attributes are never reset. All attributes are then merged and
applied to the node according to attribute precedence. When `chef-
client` finishes its run, the attributes that were applied to the node
are saved to the Chef server as part of the node object.

- A role file written in JSON, that tells `chef-client` to connect to
  the server every 5-6 minutes and reload the cookbooks:
```json
{
   "name": "web",
   "description": "Web server role.",
   "json_class": "Chef::Role",
   "default_attributes": {
     "chef_client": {
       "interval": 300,
       "splay": 60
     }
   },
   "override_attributes": {
   },
   "chef_type": "role",
   "run_list": ["recipe[chef-client::default]",
                "recipe[chef-client::delete_validation]",
                "recipe[learn_chef_apache2::default]"
   ],
   "env_run_lists": {
   }
}
```
- Upload role to the Cher server and list all to see if it got there:
```sh
$ knife role from file roles/web.json
$ knife role list
```
- Show role's details:
```sh
$ knife role show web
```
- Set the node's run-list to the ones provided by the role, and see the
  run list on the node to check if everything is set properly:
```sh
$ knife node run_list set node1-ubuntu "role[web]"
$ knife node show node1-ubuntu --run-list
```
- Then, run Chef client on the node again to apply new configuration:
```sh
$ knife ssh localhost --ssh-port 2222 'sudo chef-client' --manual-list --ssh-user vagrant --identity-file chef-server/.vagrant/machines/node1-ubuntu/virtualbox/private_key
```
- Get a brief summary of the nodes on the Chef server and the most
  recent successful run of `chef-client`:
```sh
$ knife status 'role:web' --run-list
```
- Example of a role with different run-lists for different environments:
```json
{
  "name": "webserver",
  "default_attributes": {
  },
  "json_class": "Chef::Role",
  "env_run_lists": {
    "production": [],
    "preprod": [],
    "test": [ "role[base]", "recipe[apache]", "recipe[apache::copy_test_configs]" ],
    "dev": [ "role[base]", "recipe[apache]", "recipe[apache::copy_dev_configs]" ]
    },
  "run_list": [ "role[base]", "recipe[apache]" ],
  "description": "The webserver role",
  "chef_type": "role",
  "override_attributes": {
  }
}
```


## Run-lists

A [run-list](https://docs.chef.io/run_lists.html) defines all necessary
information for Chef to configure a node in a desired state. It is an
ordered list of roles and/or recipes that are run in the exact order as
defined in the run-list.

- If a recipe apears more than once in a run-list, it will run only once
- Run-list is stored as part of node object on Chef server
- Managed by `knife` and uploaded to Chef server
- Format: `'role[NAME]'`, `'recipe[COOKBOOK::RECIPE]'`,
  `'recipe[COOKBOOK::RECIPE],COOKBOOK::RECIPE,role[NAME]'`
- Empty run-list may suggest wrong configuration settings in `knife.rb`
  or insufficient permissions for the user for Chef server and node with
  `chef-client` running on it
- Set the run-list for a node:
```sh
$ knife node run_list set test-node 'recipe[iptables]'
```
- Adding roles, recipes:
```sh
# Add a role
$ knife node run_list add NODE_NAME 'role[ROLE_NAME]'

# Add roles and recipes
$ knife node run_list add NODE_NAME \
  'recipe[COOKBOOK::RECIPE_NAME],recipe[COOKBOOK::RECIPE_NAME],role[ROLE_NAME]'

# Add a recipe with fully-qualified domain name (FQDN)
$ knife node run_list add NODE_NAME 'recipe[COOKBOOK::RECIPE_NAME]'

# Add a recipe with a cookbook
$ knife node run_list add NODE_NAME 'COOKBOOK::RECIPE_NAME'

# Add the default recipe of a cookbook
$ knife node run_list add NODE_NAME 'COOKBOOK'
```
- You can append `-a ITEM` or `--after ITEM` to add the run-list
  elements after the item in the existing run-list, or use `-b ITEM` or
  `--before ITEM` if you want to add the elements before the existing
  run-list item
- Remove roles, recipes:
```sh
# Remove a role
$ knife node run_list remove NODE_NAME 'role[ROLE_NAME]'

# Remove a recipe
$ knife node run_list remove NODE_NAME 'recipe[COOKBOOK::RECIPE_NAME]'
```
- View status of run-lists using `knife`:
```sh
$ knife status --run-list

# Query status for specific nodes
$ knife status "role:web" --run-list
```


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


## `user` resource

The [`user` resource](https://docs.chef.io/resource_user.html) lets us
add users, update existing ones, remove users, and lock or unlock their
passwords on the node.

```ruby
user 'name' do
  comment                    String # Comments about the user
  force                      TrueClass, FalseClass # force removing a user
                                # be careful if the user is logged in,
                                # also their home directory will be removed
                                # and maybe they share it with other users!
  gid                        String, Integer  # group name or ID
  home                       String # path to home directory
  iterations                 Integer # macOS only
  manage_home                TrueClass, FalseClass  # whether to do something
                                # with home directory on :create or :modify
  non_unique                 TrueClass, FalseClass  # create a duplicate user
  notifies                   # :action, 'resource[name]', :timer
  password                   String # needs libshadow-ruby1.8 to be installed
                                # shadow hash of the password
  provider                   Chef::Provider::User
  salt                       String # macOS only
  shell                      String # path to login shell executable
  subscribes                 # :action, 'resource[name]', :timer
  system                     TrueClass, FalseClass  # create a system user
  uid                        String, Integer  # user ID
  username                   String # default: 'name' if not specified
  action                     Symbol # default: :create if not specified
end
```

- Action can be one of: `:create` (default, create or update user),
  `:lock`, `:manage` (for existing user, does nothing if user does not
  exist), `:modify` (for existing user, raises an exception if user does
  not exist, `:nothing`, `:remove`, `:unlock`
- Creating a password hash:
```sh
$ mkpasswd -m sha-512

# or

$ openssl passwd -1 "theplaintextpassword"
```
- Create a user named 'random':
```ruby
user 'random' do
  manage_home true
  comment 'User Random'
  uid '1234'
  gid '1234'
  home '/home/random'
  shell '/bin/bash'
  password '$1$JJsvHslV$szsCjVEroftprNn4JHtDi'
end
```
- Create a system user with a variable:
```ruby
user_home = "/home/#{node['cookbook_name']['user']}"

user node['cookbook_name']['user'] do
  gid node['cookbook_name']['group']
  shell '/bin/bash'
  home user_home
  system true
  action :create
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
  notifies                   # :action, 'resource[name]', :timer
  provider                   Chef::Provider::Group
  subscribes                 # :action, 'resource[name]', :timer
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


## `link` resource

The [`link` resource](https://docs.chef.io/resource_link.html) lets us
define symbolic or hard links to files or directories. Remember, hard
links must refer to a file on the same file system as the file being
linked to, and it does not work with directories. By default, it creates
symbolic links, which can link to files or directories on same or
different file system.

```ruby
link 'name' do
  group                      Integer, String  # group name or ID
  link_type                  Symbol # `:symbolic` (default), `:hard`
  mode                       Integer, String  # 0755, 0400, etc.
  notifies                   # :action, 'resource[name]', :timer
  owner                      Integer, String  # user name or ID
  provider                   Chef::Provider::Link
  subscribes                 # :action, 'resource[name]', :timer
  target_file                String # default: 'name' if not specified
  to                         String # link source (what we link to)
  action                     Symbol # default: :create if not specified
end
```

- Action can be one of: `:create` (default, creates or updates a link),
  `:delete`, `:nothing`
- Create a symbolic link:
```ruby
link '/tmp/file' do
  to '/etc/file'
end
```
- Create a hard link:
```ruby
link '/tmp/file' do
  to '/etc/file'
  link_type :hard
end
```
- Delete a link only if it is symbolic:
```ruby
link '/tmp/file' do
  action :delete
  only_if 'test -L /tmp/file'
end
```
- Create platform-specific symbolic links, depending on distribution:
```ruby
include_recipe 'apache2::default'

case node['platform_family']
when 'debian'
  ...
when 'suse'
  ...
when 'rhel', 'fedora'
  ...

  link '/usr/lib64/httpd/modules/mod_apreq.so' do
    to      '/usr/lib64/httpd/modules/mod_apreq2.so'
    only_if 'test -f /usr/lib64/httpd/modules/mod_apreq2.so'
  end

  link '/usr/lib/httpd/modules/mod_apreq.so' do
    to      '/usr/lib/httpd/modules/mod_apreq2.so'
    only_if 'test -f /usr/lib/httpd/modules/mod_apreq2.so'
  end
end
```


## `package` resource

The [`package` resource](https://docs.chef.io/resource_package.html)
lets us manage packages on the node, including their installation,
upgrade, and removal.

```ruby
package 'name' do
  allow_downgrade            TrueClass, FalseClass # Yum, RPM packages
                                # only, default: false
  arch                       String, Array # Yum packages only
  default_release            String # Apt packages only (e.g. stable)
  flush_cache                Array  # [ :before, :after ]
  gem_binary                 String # Ruby gems only
  homebrew_user              String, Integer # Homebrew packages only
  notifies                   # :action, 'resource[name]', :timer
  options                    String # one or more options passed to the command
  package_name               String, Array # default: 'name' if not specified
                                # it can be an array of package names
  provider                   Chef::Provider::Package
  response_file              String # Apt packages only
  response_file_variables    Hash # Apt packages only
  source                     String # path to package in file system
  subscribes                 # :action, 'resource[name]', :timer
  timeout                    String, Integer
  version                    String, Array  # package(s)'s version to install 
  action                     Symbol # default: :install if not specified
end
```

- Action can be one of: `:install` (default), `:nothing`, `:purge`
  (Debian and Ubuntu only, remove package and its configuration files),
  `:reconfig` (requires a `response_file`), `:remove`, `:upgrade`
- Install a package:
```ruby
package 'apache2'
```
- Install a package, depending on distribution:
```ruby
package 'Install Apache' do
  case node[:platform]
  when 'redhat', 'centos'
    package_name 'httpd'
  when 'ubuntu', 'debian'
    package_name 'apache2'
  end
end
```
- Install the Bundler Ruby gem:
```ruby
gem_package 'bundler' do
  options(:prerelease => true, :format_executable => false)
end
```
- Upgrading multiple packages:
```ruby
package ['package1', 'package2']  do
  action :upgrade
end
```
- Purging multiple packages:
```ruby
package ['package1', 'package2']  do
  action :purge
end
```
- Notifications:
```ruby
package ['package1', 'package2']  do
  action :nothing
end

log 'call a notification' do
  notifies :install, 'package[package1, package2]', :immediately
end
```
- Install gem file from local file system:
```ruby
gem_package 'right_aws' do
  source '/tmp/right_aws-1.11.0.gem'
  action :install
end
```
- Install packages with specific versions:
```ruby
package 'tar' do
  version '1.16.1-1'
  action :install
end

package 'mysql-server' do
  version node['mysql']['version']
  action :install
end
```
- Install a package with options:
```ruby
package 'debian-archive-keyring' do
  action :install
  options '--force-yes'
end
```
- Install `sudo` and configure `/etc/sudoers`:
```ruby

package 'sudo' do
  action :install
end

if node['authorization']['sudo']['include_sudoers_d']
  directory '/etc/sudoers.d' do
    mode        '0755'
    owner       'root'
    group       'root'
    action      :create
  end

  cookbook_file '/etc/sudoers.d/README' do
    source      'README'
    mode        '0440'
    owner       'root'
    group       'root'
    action      :create
  end
end

template '/etc/sudoers' do
  source 'sudoers.erb'
  mode '0440'
  owner 'root'
  group platform?('freebsd') ? 'wheel' : 'root'
  variables(
    :sudoers_groups => node['authorization']['sudo']['groups'],
    :sudoers_users => node['authorization']['sudo']['users'],
    :passwordless => node['authorization']['sudo']['passwordless']
  )
end
```
- Use `case` statement to specify platform:
```ruby
package 'curl'
  case node[:platform]
  when 'redhat', 'centos'
    package 'package_1'
    package 'package_2'
    package 'package_3'
  when 'ubuntu', 'debian'
    package 'package_a'
    package 'package_b'
    package 'package_c'
  end
end
```
- Use array to specify an action on several packages at once:
```ruby
%w{package-a package-b package-c package-d}.each do |pkg|
  package pkg do
    action :upgrade
  end
end
```
- Use a hash to install several gems with specific versions at once:
```ruby
{"fog" => "0.6.0", "highline" => "1.6.0"}.each do |g,v|
  gem_package g do
    version v
  end
end
```
