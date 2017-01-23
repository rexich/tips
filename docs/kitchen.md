# Kitchen 101: Using Kitchen for Continuous Integration with Chef

Kitchen is used for continuous integration testing of Chef recipes in a
Vagrant environment.

In the directory where the Chef repo is located, we issue:

```sh
$ kitchen init --driver=kitchen-vagrant
```

Kitchen will create its configuration files and set things up for the
testing. You might need to issue the command with `sudo` prepended, in
order to install the necessary driver gems. The file `.kitchen.yml` in
the root of the Chef repo contains the configuration for Kitchen:

```yaml
---
driver:
  name: vagrant

provisioner:
  name: chef_solo

platforms:
  - name: ubuntu-12.04
  - name: centos-6.4

suites:
  - name: default
    run_list:
      - recipe[git::default]
    attributes:
```

- `driver.name` defines the component used to create a virtual
machine for testing the recipe, `vagrant` is considered default unless
specified otherwise here
- `provisioner.name` tells which software to use for provisioning, in
this case that would be `chef-solo`
- `platforms` has one or more `name` parts that define the Linux
distributions and their versions that will be used as a base system for
performing the tests in a virtual machine
- `suites` defines what we want to test, and here we define its `name`,
  the `runlist`s to run, and `attributes` for specifying test node's
  attributes, and each suite will run on the platforms specified

After editing the configuration file, we can check Kitchen's suites:

```sh
$ kitchen list
Instance             Driver   Provisioner  Verifier  Transport  Last Action    Last Error
default-ubuntu-1404  Vagrant  ChefSolo     Busser    Ssh        <Not Created>  <None>
```

The `Instance` is the name of the suite we want to run. Issue this
command to create an instance and prepare a Vagrant box for Kitchen:

```sh
$ kitchen create default-ubuntu-1404
-----> Starting Kitchen (v1.15.0)
-----> Creating <default-ubuntu-1404>...
       Bringing machine 'default' up with 'virtualbox' provider...
       ==> default: Importing base box 'bento/ubuntu-14.04'...
==> default: Matching MAC address for NAT networking...
       ==> default: Checking if box 'bento/ubuntu-14.04' is up to date...
       ==> default: Setting the name of the VM: kitchen-git-cookbook-default-ubuntu-1404_default_1485160924636_5827
       ==> default: Clearing any previously set network interfaces...
       ==> default: Preparing network interfaces based on configuration...
           default: Adapter 1: nat
       ==> default: Forwarding ports...
           default: 22 (guest) => 2222 (host) (adapter 1)
       ==> default: Booting VM...
       ==> default: Waiting for machine to boot. This may take a few minutes...
           default: SSH address: 127.0.0.1:2222
           default: SSH username: vagrant
           default: SSH auth method: private key
           default: 
           default: Vagrant insecure key detected. Vagrant will automatically replace
           default: this with a newly generated keypair for better security.
           default: 
           default: Inserting generated public key within guest...
           default: Removing insecure key from the guest if it's present...
           default: Key inserted! Disconnecting and reconnecting using new SSH key...
       ==> default: Machine booted and ready!
       ==> default: Checking for guest additions in VM...
       ==> default: Setting hostname...
       ==> default: Mounting shared folders...
           default: /tmp/omnibus/cache => /home/rex/.kitchen/cache
       ==> default: Machine not provisioned because `--no-provision` is specified.
       [SSH] Established
       Vagrant instance <default-ubuntu-1404> created.
       Finished creating <default-ubuntu-1404> (0m39.40s).
-----> Kitchen is finished. (0m39.52s)
```

A Vagrant box is prepared, up and running. Let's list Kitchen's suites:

```sh
$ kitchen list
Instance             Driver   Provisioner  Verifier  Transport  Last Action  Last Error
default-ubuntu-1404  Vagrant  ChefSolo     Busser    Ssh        Created      <None>
```

After we write our cookbooks and recipes, we can run the tests for the
Ubuntu 14.04 Vagrant box:

```sh
$ kitchen converge default-ubuntu-1404
```

What happens? Chef gets installed on the Vagrant box using the Omnibus
installer, cookbooks and recipes and a minimal `chef-solo` configuration
are uploaded on the box, and a Chef run is performed using the runlist
and attributes specified in the `.kitchen.yml` configuration file.

Kitchen's exit code 0 means tests completed successfully, any other
value indicates an error.

Checking the Kitchen's suites again, we see a change in Last Action:

```sh
$ kitchen list
Instance             Driver   Provisioner  Verifier  Transport  Last Action  Last Error
default-ubuntu-1404  Vagrant  ChefSolo     Busser    Ssh        Converged    <None>
```

Manual verification can be performed by logging into the Vagrant box,
and Kitchen provides such a facility as well:

```sh
$ kitchen login default-ubuntu-1404
```

The reason we use Kitchen is to write and perform tests, so we can check
if the Chef run will complete successfully and the system will be in a
defined and expected state. Busser (the `Verifier`) is a component used
to perform tests using BATS (Bash Automated Test Suite). We create a
directory for these tests using:

```sh
$ mkdir -p test/integration/default/bats
```

The `default/` directory corresponds to the name of the suite we want to
execute as defined in the `.kitchen.yml` file, in `suites.name`, and the
`bats/` directory tells Kitchen we want to use BATS testing suite. In
that directory, we can write a simple test called `git_installed.bats`:

```sh
#!/usr/bin/env bats

@test "git binary is found in PATH" {
  run which git
  [ "$status" -eq 0 ]
}
```

Then we use Kitchen to verify the test, and we check the return value to
make sure it is zero, which confirms that the test is verified and is
working well:

```sh
$ kitchen verify default-ubuntu-1404
-----> Starting Kitchen (v1.15.0)
-----> Verifying <default-ubuntu-1404>...
       Preparing files for transfer
-----> Busser installation detected (busser)
       Installing Busser plugins: busser-bats
       Plugin bats already installed
       Removing /tmp/verifier/suites/bats
       Transferring files to <default-ubuntu-1404>
-----> Running bats test suite
 ✓ git binary is found in PATH
       
       1 test, 0 failures
       Finished verifying <default-ubuntu-1404> (0m2.71s).
-----> Kitchen is finished. (0m2.83s)
$ echo $?
0
```

Then, listing the Kitchen suites will confirm the verification:

```sh
$ kitchen list
Instance             Driver   Provisioner  Last Action
default-ubuntu-1204  Vagrant  ChefSolo     Verified
```

Once we confirm the `Last Action` says `Verified`, which shows our test
is correctly written, we can commence with the Kitchen testing:

```sh
$ kitchen test default-ubuntu-1404
```

First, it will destroy any active Vagrant box instance of the test, then
it will create a Vagrant box, install and configure Chef on it, upload
the cookbooks and tests, run the Chef runlist and perform the actions,
run the tests, return the results, and then destroy the Vagrant box, in
this order: `destroy → create → converge → setup → verify → destroy`.

Since when destroying you need to wait for the Vagrant box to be created
again, and that takes time, you can use `kitchen converge` and
`kitchen verify` instead while working on your recipes and tests, and
`kitchen test` when performing the final testing.

Destroying instances can be done using:

```sh
$ kitchen destroy default-ubuntu-1404
```

If you don't specify an instance name, an action will be performed for
all of the instances defined in the `suites` section of `.kitchen.yml`.


## Configuration

Your project's Kitchen configuration is stored in the root directory
of the project in a file called `.kitchen.yml`. A global configuration
file that will apply to all projects can be stored in you home directory
in the file called `$HOME/.kitchen/config.yml`.

You can override these files by setting environmental variables, and
also settings are loaded and merged in this order or precedence:

```sh
export KITCHEN_LOCAL_YAML=/path/to/your/local/.kitchen.local.yml
export KITCHEN_PROJECT_YAML=/path/to/your/project/.kitchen.yml
export KITCHEN_GLOBAL_YAML=/path/to/your/global/config.yml
```

## Multiple suites

A `.kitchen.yml` configuration file can look like this:

```yaml
---
driver:
  name: vagrant

provisioner:
  name: chef_solo

platforms:
  - name: ubuntu-14.04
  - name: ubuntu-10.04
  - name: centos-7.3
suites:
  - name: default
    run_list:
      - recipe[git::default]
    attributes:
  - name: server
    run_list:
      - recipe[git::server]
    attributes:
```

There are two suites specified: `default` and `server`. Each of these
configures a specific case in isolation and convergence and verification
can be performed upon them. Let's list the Kitchen's suites:

```sh
$ kitchen list
Instance             Driver   Provisioner  Verifier  Transport  Last Action    Last Error
default-ubuntu-1404  Vagrant  ChefSolo     Busser    Ssh        <Not Created>  <None>
default-ubuntu-1004  Vagrant  ChefSolo     Busser    Ssh        <Not Created>  <None>
default-centos-73    Vagrant  ChefSolo     Busser    Ssh        <Not Created>  <None>
server-ubuntu-1404   Vagrant  ChefSolo     Busser    Ssh        <Not Created>  <None>
server-ubuntu-1004   Vagrant  ChefSolo     Busser    Ssh        <Not Created>  <None>
server-centos-73     Vagrant  ChefSolo     Busser    Ssh        <Not Created>  <None>
```

For each suite defined, we get instances for each configuration, too.

To exclude a platform from a suite, add `excludes:` and name them, so
Kitchen will not create instances for them:

```yaml
---
driver:
  name: vagrant

provisioner:
  name: chef_solo

platforms:
  - name: ubuntu-14.04
  - name: ubuntu-10.04
  - name: centos-7.3
suites:
  - name: default
    run_list:
      - recipe[git::default]
    attributes:
  - name: server
    run_list:
      - recipe[git::server]
    attributes:
    excludes:
      - ubuntu-10.04
      - centos-7.3
```


## Serverspec testing and Berkshelf cookbook dependencies solver

[Serverspec](http://serverspec.org/) is a RSpec-based testing suite for
writing tests that target continuous integration of servers. Kitchen
supports Serverspec fully. To write a test, first we need to make a
directory in project's root called `test/integration/server/serverspec`:

```sh
$ mkdir -p test/integration/server/serverspec
```

Here, we are creating a `serverspec` directory for the `server` suite.
A Serverspec test is a Ruby file, and in our case we'll name it
`git_daemon_test.rb`, and its contents is the following code:

```ruby
require "serverspec"

# Required by Serverspec
set :backend, :exec

describe "Git Daemon" do

  it "is listening on port 9418" do
    expect(port(9418)).to be_listening
  end

  it "has a running service of git-daemon" do
    expect(service("git-daemon")).to be_running
  end

end
```

In Serverspec (and RSpec), we `describe` a component, and what does `it`
do, and what do we `expect` it to do. These are the test expressions.

The Serverspec test will fail unless we write a new recipe, stored in
`recipes/server.rb`:

```ruby
include_recipe "git"
include_recipe "runit"

package "git-daemon-run"

runit_service "git-daemon" do
  sv_templates false
end
```

Likewise, this recipe depends on the recipes `runit` and `git`, and they
are its *dependencies*. For that, we need to edit the `metadata.rb` file
and add the dependency:

```ruby
name "git"
version "0.1.0"

depends "runit", "~> 1.4.0"
```

Once we defined the dependency, we'll need a tool to resolve it and
download the missing cookbooks and recipes for us. For that purpose we
will use [Berkshelf](https://docs.chef.io/berkshelf.html) to fetch and
install `runit`. Let's install Berkshelf, which is a Ruby gem:

```sh
$ sudo gem install berkshelf
```

Then we write the `Berksfile` in the root of our project's directory:

```ruby
source "https://api.berkshelf.com"

metadata
```

The `metadata` directive means Berkshelf will use our `metadata.rb` file
to find the necessary cookbooks and fetch them. Now we can perform the
verification of our Serverspec tests:

```sh
$ kitchen verify server-ubuntu-1404
```

Note: In case you get complaints that `thor` version creates conflicts,
remove it with `sudo gem uninstall thor --version 0.19.4`, and try the
verification again, it will work.

All tests should pass. The output of listing Kitchen's suites should be
similar to this:

```sh
$ kitchen list
Instance             Driver   Provisioner  Verifier  Transport  Last Action    Last Error
default-ubuntu-1404  Vagrant  ChefSolo     Busser    Ssh        <Not Created>  <None>
default-ubuntu-1004  Vagrant  ChefSolo     Busser    Ssh        <Not Created>  <None>
default-centos-73    Vagrant  ChefSolo     Busser    Ssh        <Not Created>  <None>
server-ubuntu-1404   Vagrant  ChefSolo     Busser    Ssh        Verified       <None>
server-ubuntu-1004   Vagrant  ChefSolo     Busser    Ssh        <Not Created>  <None>
server-centos-73     Vagrant  ChefSolo     Busser    Ssh        <Not Created>  <None>
```


## Extras

To change the user name to access the running Vagrant instance, add to
  your `.kitchen.yml` before running Kitchen:

```yaml
yaml transport: username: ubuntu
```

Configuring Vagrant is possible from the `.kitchen.yml` file:

```yaml
name: vagrant
network:
  - ["forwarded_port", {guest: 80, host: 8080, auto_correct: true}]
  - ["forwarded_port", {guest: 443, host: 8443}]
  - ["private_network", {ip: "10.0.0.1"}]
customize:
  cpus: 2
  memory: 4096
```

The necessary `Vagrantfile` will be generated out of this configuration
for the purpose of creating instances and performing the tests. Refer to
the [plugin's web page](https://github.com/test-kitchen/kitchen-vagrant)
for more options.
