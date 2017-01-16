# Chef

## Installing
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

- Chef updates what is necessary and changed, and makes sure it gets to
  a consistent state defined in the recipes - *set and forget* principle
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
- `knife` is a [tool](https://docs.chef.io/knife.html) that provides an
  interface between local chef-repo and the Chef server, used to manage:
  - nodes, cookbooks, recipes, roles
  - data bags (stores of JSON data, including encrypted ones)
  - environments, cloud resources (including provisioning)
  - installation of `chef-client` on management workstations
  - searching of indexed data on the Chef server
