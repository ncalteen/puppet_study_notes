# Using Modules

- It is very likely somebody has already written the module you need.

## Finding Modules

### Puppet Forge

- https://forge.puppet.com
- Search modules created by others.
- Default repository used by the `puppet module` command.
- Can be searched from the console as well as the CLI.

  ```bash
  puppet module search apache
  ```

### Internal Repositories

- You can use the `puppet module` command with any forge (internal or external).

  ```bash
  puppet module search --module_repository=http://forge.example.com apache
  ```

- If you exclusively use an internal forge, you can add this to `puppet.conf`.

  **/etc/puppetlabs/puppet/puppet.conf**

  ```ini
  [main]
  module_repository = http://forge.example.com
  ```

## Evaluating Module Quality

### Puppet Supported

- Written and officially supported by Puppet.
- Tested with Puppet Enterprise.
- Subject to PE Support.
- Maintained by Puppet.

### Puppet Approved

- Reviewed by Puppet for quality standards.
- Developed in accordance with module best practices.
- Adheres to Puppet's style guidelines.
- Well documented.
- Maintained and versioned.

## Installing Modules

### Installing from a Puppet Forge

- Simple to install from the CLI.

  ```bash
  sudo puppet module install puppetlabs-stdlib
  ```

  - Installs by default to `/etc/puppetlabs/code/environments/production/modules`.
  - Install to another environment with the `--environment` flag.
- To migrate a module to another environment, you can run `puppet module install` again, or simply copy the module over.

### Installing from GitHub

- Make sure `git` is installed.

  ```bash
  sudo yum install -y git
  ```

- You can then clone from any available Git repository.

  ```bash
  cd /etc/puppetlabs/code/environments/test/modules
  git clone https://github.com/jorhett/puppet-mcollevtive mcollective
  ```

## Testing a Single Module

- In most situations, you will need to:
  - Declare the module's classes in the node definition.
  - Define Hiera data keys under the module's name.
- Intall and configure the `puppetlabs-ntp` module.

  ```bash
  cd /etc/puppetlabs/code/environments/test/modules
  sudo puppet module install --modulepath=. puppetlabs-ntp
  ```

  - The install command automatically installs dependencies.
- This module can be installed without any input data, using default values.

  ```bash
  sudo puppet apply --environment test --execute 'include ntp'
  ```

## Defining Config with Hiera

- Depending how a module is set up, you canalter its functionality by setting data in Hiera under the class name.
- __Example:__ The `puppetlabs-ntp` module can be set to specify which interfaces will accept connections, and which systems can connect.
  - To provide data for a module's input, it must be named `[module]::[parameter]`.

    **/etc/puppetlabs/code/hieradata/common.yaml**

    ```yaml
    ---
    ntp::interfaces:
      - '127.0.0.1'
    ntp::restrict:
      - 'default kod nomodify notrap nopeer noquery'
      - '-6 default kod nomodify notrap nopeer noquery'
      - '127.0.0.1'
      - '-6 ::1'
      - '192.168.250.0/24'
      - '-6 fe80::'
    ```

  - Make use of host-level overrides by adding client-specific data.

    **/etc/puppetlabs/code/hieradata/hostname/client.yaml**

    ```yaml
    ---
    ntp::interfaces:
      - '127.0.0.1'
      - '192.168.250.10'
    ```

    ```bash
    sudo puppet apply --environment test --execute 'include ntp'
    ```

  - Verify the client-specific data was used.

    ```bash
    grep 192.168.250 /etc/ntp.conf
    grep listen /etc/ntp.conf
    ```

## Assigning Modules to Nodes

- Best-practice method to assign module classes to a node is to define the classes within Hiera data.

  **/etc/puppetlabs/code/environments/test/manifests/site.pp**

  ```puppet
  # Look up all classes defined in Hiera and other data sources.
  lookup('classes', Array[String], 'unique').include
  ```

### Using Hiera for Module Assignment

  **/etc/puppetlabs/code/environments/production/manifests/site.pp**

  ```puppet
  lookup('classes', Array[String], 'unique').include
  ```

- With this added, we can assign modules using Hiera data.

### Assigning Classes to Every Node

- This ca be assigned in `common.yaml` using a top-level key named 'classes'.

  **/etc/puppetlabs/code/hieradata/common.yaml**

  ```yaml
  ---
  classes:
    - 'ntp' # This will apply the ntp class to all nodes going forward.
  ```

  ```bash
  sudo puppet apply --environment test /etc/puppetlabs/code/environments/test/manifests/
  ```

  - The first time this is run, nothing is changed (we already ran the ntp module).

    ```bash
    sudo systemctl stop ntpd
    sudo puppet apply --environment test /etc/puppetlabs/code/environments/test/manifests/
    ```

### Altering the Class List Per Node

- __Example:__ You may want to run Puppet agent on every node, but Puppet server only on the master.
  - You can define this with global and node-specific data.

    **/etc/puppetlabs/code/hieradata/common.yaml**

    ```yaml
    ---
    classes:  # Applied to all nodes
      - ntp
      - puppet::agent
    ```

    **/etc/puppetlabs/code/hieradata/hostname/puppetserver.yaml**

    ```yaml
    ---
    classes:
      - puppet::server
    ```
- Class assignment is always done as an array merge (removing duplicates).

### Avoiding Node Assignments in Manifests

- In Puppet v2 and older, node assignments could only be done in `site.pp`.
- Node assignment via `site.pp` is still possible, but inheritance was removed.

  ```puppet
  # OLD Method
  node 'webserver' inherits web-server {
      class { 'apache':
        modules => 'fail2ban',
      }
  }
  node 'passenger' inherits web-server {
      class { 'apache':
        modules => 'passenger',
      }
  }
  ```

  ```yaml
  # NEW Method
  # common.yaml
  classes:
    - apache

  # webserver.yaml
  apache::modules:
    - fail2ban

  # passenger.yaml
  apache::modules:
    - passenger
  ```

## Examining a Module

- __OS Support:__ Will the module support your intended OS(es)?
- __Module Namespace:__ Does the module or any of its dependencies conflict with a same-named module already in use?
- __Environment Assumptions:__ Will the module enforce local assumptions not supported in your environment?
- __Sloppy Code:__ Is the module readable/understandable?
  - Does the module overwrite or require global variables?
- __Resource Namespace:__ Does the module use resource titles that conflict with resources already in use?
- __Greedy Collectors:__ Does the module use collectors that accidentally grab other resources?