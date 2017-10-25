# Separating Data from Code

- A module written for a single node with hardcoded data may work, but it is not portable or practical.

## Introducing Hiera

- Key/value lookup tool for configuration data.
- Dynamically look up configuration data for Puppet manifests.
- Utilizes a configurable hierarchy of information, and can be tuned for how information is structured in your organization.
  - __Example:__ An organization may structure data this way:

    1. Company-wide common data.
    1. Operating system specific changes.
    1. Site specific information.

  - This can be used to merge common data with node and environment specific overrides.

## Creating Hiera Backends

- Hiera supports JSON and YAML data file backends.
- Hiera supports 5 data types:
  - String
  - Number
  - Boolean
  - Array
  - Hash

### Hiera Data in YAML

- Must have a `.yaml` extension.
- Must start with three dashes on the first line ('---').
- Use spaces, not tabs, for indentation.

  ```yaml
  puppet:
    ensure: 'present' # String
    version: '4.4.0'
    agent:            # Hash
      running: 'running'
      atboot: true
    components:       # Array
      - 'facter'
      - 'puppet'
  ```

### Hiera Data in JSON

- Must have a `.json` file extension.
- The root of each data source must be a single hash enclosed in brackets ('{ ... }').

  ```json
  {
      "puppet": {
          "ensure": "present",
          "version": "4.4.0",
          "agent": {
              "running": "running",
              "atboot": true
          },
          "components": [
              "facter",
              "puppet"
          ]
      }
  }
  ```

- Unlike YAML, JSON does not support comments.

### Puppet Variable and Function Lookup

- You can look up Puppet variables or execute functions to interpolate data within a Hiera value.
  - Performed on any value prefixed by '%' and enclosed in '{ ... }'.
  - __Example:__

    ```puppet
    %{facts.hostname}
    %{split([1, 2, 3])}
    ```

## Configuring Hiera

- Puppet looks at the config file specified by the `hiera_config` configuration variable.
  - By default, this is `${codedir}/hiera.yaml`, or `/etc/puppetlabs/code/hiera.yaml`.
- The Hiera config file is a YAML hash.
  - Items at the top are global settings.
- All settings are optional and fall back to default values.

### Backends

- Lists the backend data providers that Hiera should use.
- Supports JSON, YAML, or custom providers (not covered here).

  ```yaml
  :backends:
    - yaml
    - json
  ```

### Backend Configuration

- For every provider in the `:backends` array, a separate global setting should be provided.
- The only required key is `:datadir`, which specifies the directory for data files.

  ```yaml
  :yaml:
    :datadir: /etc/puppetlabs/code/environments/%{::environment}/hieradata
  ```

- You can use the same directory for each data source (keeping note of file extensions).

### Logger

- By default, Hiera command line tools log warning/debug messages to STDOUT.
- The `:logger` configuration value allows you to change this.
  - __console:__ Emit warnings/debugs to STDOUT (default).
  - __puppet:__ Send messages to Puppet's logging system.
  - __noop:__ Don't emit messages.
  - __name:__ Use the Ruby class Hiera::<name>_logger.
    - Must provide warn/debug class methods, which accept a single string.
- This is only for command line tools, as Puppet uses the internal logger.

### Hierarchy

- The `:hierarchy` defines the priority order for lookup of configuration data.
- For single values, Hiera will proceed through the hierarchy until it finds the value (at which point it will stop).
- For arrays and hashes, Hiera will merge data from each level of the hierarchy as configured by the merge strategy.
- __Static Data Sources:__ Files explicitly named inthe hierarchy that contain data.
- __Dynamic Data Sources:__ Files that are named using the interpolation of local config data, such as hostname, OS, or other info on the node.
- __Recommended Strategy:__
  - Put default values in a file named `common.yaml`.
  - Put all OS-specific info in a file named for the OS family as returned by Facter (`RedHat.yaml`, `Debian.yaml`, etc.).
  - Put node-specific information in a file named with the node's FQDN (`[FQDN].yaml`).

    ```yaml
    :hierarchy:
      - "fqdn/%{facts.fqdn}"   # Node-specific
      - "os/%{facts.osfamily}" # OS-family specific
      - common                 # Everything else
    ```

- If you have multiple backends configured, Hiera will evaluate the entire hierarchy for each backend, in the order listed.

### Merge Strategy

- In previous versions, you could only set merge strategy globally, using `:merge_behavior` in `hiera.yaml`.
- __Supported Merge Strategies:__
  - __first (default):__ Returns the first value found, with no merging.
    - Keys from the higher priority will win.
  - __hash:__ Merge keys only.
    - Values from the higher priority match will be used exclusively, without any values from the lower priority.
  - __deep:__ Recursively merge array and hash values.
    - Lower-priority values that don't conflict will be merged with higher-priority values.
  - __unique:__ Flatten array and scalar values form all priorities into a single array.
    - Duplicate values will be dropped.
    - Hashes will cause an error.
- Merge strategy can be set on a per-key basis.
  - Set an entry in the `lookup_options` hash.
    - A key in this hash can be named for the value.
    - Defined in global, environment, or module data.
    - The hash of values assigned to this key will set the lookup options and merge strategy.
  - Set options when calling the `lookup()` function.
    - These options will override any set in data.

### Complete Example

- Enable YAML data input from `/etc/puppetlabs/code/hieradata`.
  - Allows you to share Hiera data between environments.

    **/etc/puppetlabs/code/hiera.yaml**

    ```yaml
    ---
    version: 5
    defaults:        # for any hierarchy level without these keys
      datadir: /etc/puppetlabs/code/hieradata  # directory name inside the environment
      data_hash: yaml_data

    hierarchy:
      - name: "Hostname"
        path: "hostname/%{trusted.hostname}.yaml"

      - name: "OS-specific values"
        path: "os/%{facts.osfamily}.yaml"

      - name: "common"
        path: "common.yaml"
    ```

## Looking Up Hiera Data

  ```bash
  mkdir -p /etc/puppetlabs/code/hieradata/hostname
  ```

  **/etc/puppetlabs/code/hieradata/common.yaml**

  ```yaml
  ---
  puppet::status: 'running'
  puppet::enabled: true
  ```

  **/etc/puppetlabs/code/hieradata/hostname/client.yaml**
  
  ```yaml
  ---
  puppet::status: 'stopped'
  puppet::enabled: false
  ```

### Checking Hiera Values from the Command Line

- The Hiera command line tool doesn't have facts and other config data from Puppet.

  ```bash
  $ hiera puppet::status
  running
  ```

- Without facts, Hiera will return only values it knows how to find.
  - In this case, it cannot look up the hostname of the node to check `client.yaml`.
- Puppet's `lookup()` function will include Facter data.

  ```bash
  puppet apply -e "notice(lookup('puppet::enabled'))"
  # or
  puppet lookup puppet::status
  ```

### Performing Hiera Lookups in a Manifest

  **/vagrant/manifests/hierasample.pp**

  ```puppet
  # Always set a default value
  $status  = lookup({ name => 'puppet::status',  default_value => 'running' })
  $enabled = lookup({ name => 'puppet::enabled', default_value => true })

  notify { 'puppet-settings':
    message => "Status should be ${status}, start at boot ${enabled}.",
  }

  # Now the same code can be used regardless of the value
  service { 'puppet':
    ensure => $status,
    enable => $enabled,
  }
  ```

  ```bash
  sudo puppet apply /vagrant/manifests/hierasample.pp
  rm /etc/puppetlabs/code/hieradata/hostname/client.yaml
  sudo puppet apply /vagrant/manifests/hierasample.pp
  ```

### Testing Merge Strategy

- Define some users in `common.yaml`, as well as a higher priority hash in `client.yaml`.

  **common.yaml**

  ```yaml
  users:
    jill:
      uid: 1000
      home: '/home/jill'
    jack:
      uid: 1001
      home: '/home/jack'
  ```

  **client.yaml**

  ```yaml
  users:
    jill:
      uid: 1000
      home: '/home/jill'
    jack:
      uid: 1001
      home: '/home/jack'
    janet:
      uid: 1002
      home: '/home/janet'
  ```

- Lookup users with no overrides set.

  ```bash
  puppet lookup users
  ```

- Test the lookup function from other nodes.

  ```bash
  puppet lookup --node default users
  ```

- Test a recursive merge through both hashes to find unique keys.

  ```bash
  puppet lookup users --merge deep
  ```

  - When keys match, the higher priority value is chosen.
- We can also set lookup_options for this key to specify a deep merge.

  **common.yaml**
  lookup_options:
    users:
      merge: deep