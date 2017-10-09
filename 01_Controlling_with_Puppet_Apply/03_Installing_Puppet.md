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