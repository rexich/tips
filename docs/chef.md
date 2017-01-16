# Chef

# Installing
Chef client:
```sh
$ curl -L https://omnitruck.chef.io/install.sh | sudo bash
```

Chef Development Kit:
```sh
$ curl -L https://omnitruck.chef.io/install.sh | sudo bash -s -- \
-P chefdk -c stable
```


# Usage

- Run a recipe: `chef-client --local-mode hello.rb`
- Another way: `chef-apply recipe.rb`
- `chef-solo` executes `chef-client` in a way that does not require
  Chef server to converge cookbooks, runs in local mode, has no
  centralized distribution of cookbooks or API, no authentication, no
  authorization, can be run as a daemon; `chef-solo -c ~/chef/solo.rb`
- [Chef-solo](https://docs.chef.io/chef_solo.html) has `solo.rb` as its
  configuration file, where you can set paths and recipes to run
