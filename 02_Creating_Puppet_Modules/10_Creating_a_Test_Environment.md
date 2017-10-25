# Creating a Test Environment

- Environments are used to serve agents different versions of modules and data.
- Primary use is to enable testing of catalog changes without breaking production.
- Enabled by default in Puppet 4.

## Verifying the Production Environment

- Production is the default environment used by agents.

  ```bash
  ls -l /etc/puppetlabs/code/environments/production
  ```

- Includes an environment configuration file, and directories for Hiera data, modules, and manifests.

## Creating the Test Environment

- You will need to create the necessary directory structure to support a new Puppet environment.

  ```bash
  mkdir -p /etc/puppetlabs/code/environments/test/{modules,hieradata,manifests}
  ```

- Make a reminder that you are in a test environment.

  **/etc/puppetlabs/code/environments/test/manifests/site.pp**

  ```puppet
  notify { 'usingtest':
    message => 'Processing catalog from the test environment.',
  }
  ```

## Changing the Base Module Path

- Modules can be shared across multiple environments.

  **/etc/puppetlabs/puppet.conf**

  ```ini
  [main]
  environmentpath = /etc/puppetlabs/code/environments # Default value.
  basemodulepath = /etc/puppetlabs/code/modules # Default value.
  ```

- The `basemodulepath` property acts as a fallback location for modules not found in the environment's `modules/` directory.
  - Allows you to share common, well-tested modules between environments.