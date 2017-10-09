# Installing Puppet

- The latest Puppet Collection is available via the puppetlabs repository.

  ```bash
  sudo yum install -y \
  > http://yum.puppetlabs.com/puppetlabs-release-pc1-el-7.noarch.rpm
  # puppetlabs-release-[collection]-[os_abbreviation]-[os-version]
  sudo yum repolist
  ```

## What is a Puppet Collection

- The Puppet ecosystem contains many related packages.
  - Puppet
  - Facter
  - MCollective
  - Ruby Interpreter
- Puppet agent, server, and PuppetDB are self-standing but interdependent applications.
- Puppet and all core dependencies are shipped in a single package.
- Components of the Puppet ecosystem are tested and shipped together.
  - All versions in a single collection are guaranteed to work together.

## Installing the Puppet Agent

- Puppet commands are not installed in `/usr/bin`.

  ```bash
  sudo yum install -y puppet-agent
  ```

## Reviewing Dependencies

- Puppet 4+ uses an all-in-one installer.
- The puppet-agent package includes `/etc/profile.d/puppet-agent.sh`, which adds `/opt/puppetlabs/bin` to your path.
- If you reset your $PATH config files, remember to add:

  ```bash
  source /etc/profile.d/puppet-agent.sh
  ```

- __Facter__: Evaluates a system and provides a number of facts about it.
  - Any node-specific or custom information.

  ```bash
  facter ipaddress # facter [attribute]
  ```

- __Hiera__: A component used to load data used by Puppet manifests and modules.
  - Allows you to provide default values and override or expand them through a customizable hierarchy.

- __Marionette Collective (Mollective)__: An orchestration framework integrated with Puppet.

## Making Tests Convenient

- When running Puppet with sudo, it uses system-level config files:

  ```bash
  $ sudo puppet config print | grep dir
  confdir = /etc/puppetlabs/puppet
  codedir = /etc/puppetlabs/code
  vardir = /opt/puppetlabs/puppet/code
  logdir = /var/log/puppetlabs/puppet
  # ...
  ```

- When running Puppet without sudo, it will use paths in your home directory.

  ```bash
  $ puppet config print | grep dir
  confdir = ~/.puppetlabs/etc/puppet
  codedir = ~/.puppetlabs/etc/code
  vardir = ~/.puppetlabs/opt/puppet/cache
  logdir = ~/.puppetlabs/var/log
  # ...
  ```

- Either use sudo every time Puppet is run, or set up a config gile to use system paths.

## Running Puppet without sudo

  ```bash
  $ cat ~/.puppetlabs/etc/puppet/puppet.conf
  # Allow "puppet hiera" and "puppet module" without sudo
  [main]
    logdest = console
    confdir = /etc/puppetlabs/puppet
    codedir = /etc/puppetlabs/code
  ```

## Running Puppet with sudo

- You will need to add `/opt/puppetlabs/bin` to sudo's secure_path defaults.

  ```bash
  sudo grep secure_path /etc/sudoers \
  > | sed -e 's#$#:/opt/puppetlabs/bin#' \
  > | sudo tee /etc/sudoers.d/puppet-securepath.sh
  ```