# Extending Modules with Plugins

- Plugins are used to provide new facts, functions, and module-specific data that can be used in the Puppet catalog.
- Provide new customized facts that can be used within your module.
  - Facts can be supplied by a program that outputs one fact per line.
  - Facts can be supplied by Ruby functions.
  - Facts can be read from YAML, JSON, or text files.
- Provide new functions that extend Puppet.
  - Facts can be written in either Puppet or Ruby languages.
- Provide data lookups that are environment or module specific.
  - Custom data sources are written in Ruby.

## Adding Custom Facts

- Custom facts extend the built-in facts that Facter provides with node-specific values useful for the module.
- Module facts are synced from the module to the agent, and are available for use during catalog compilation.
- In Puppet 4, custom facts can return any of the Scalar data types (String, Numeric, Boolean, etc.), as well as any Collection type (Array, Hash, Struct, etc.).

### External Facts

- There are two ways to provide external facts:
  - Write fact data out in YAML, JSON, or text format.
  - Provide a program or script to output fact names and values.
    - Must be executable by the Puppet agent.

#### Structured Data

- You can place structured data files in the `facts.d/` directory of your module.
- Must be in a known format, and must be named with the file extension for their format (`.json`, `.yaml`, `.txt`).
- __Example:__ Three facts written in YAML.

  ```yaml
  ---
  my_fact: 'myvalue'
  my_effort: 'easy'
  is_dynamic: 'false'
  ```

- The text format uses a single `key=value` pair on each line of the file.
  - This only supports string values (no arrays or hashes).

  ```text
  my_fact=myvalue
  my_effort=easy
  is_dynamic=false
  ```

#### Programs

- Executable programs or scripts can also be placed in `facts.d/` of your module.
- The executable must have the execute bit set for root (or the user running Puppet).
- Each fact must be output as `key=value` on its own line.

  **/etc/puppetlabs/code/environments/test/modules/puppet/facts.d/three_simple_facts.sh**

  ```bash
  #!/bin/bash
  echo "my_fact=myvalue"
  echo "my_effort=easy"
  echo "is_dynamic=false"
  ```

- Test this on your client node.

  ```bash
  $ chmod 0755 /etc/puppetlabs/code/environments/test/modules/puppet/facts.d/three_simple_facts.sh
  $ /etc/puppetlabs/code/environments/test/modules/puppet/facts.d/three_simple_facts.sh
  my_fact=myvalue
  my_effort=easy
  is_dynamic=false
  ```

#### Facts on Windows

- Requires the same execute permissions and output format.
- The program or script must be written with a known extension.
  - `.com`
  - `.exe`
  - `.ps1`
  - `.cmd`
  - `.bat`

  ```powershell
  Write-Host "my_fact=myvalue"
  Write-Host "my_effort=easy"
  Write-Host "is_dynamic=false"
  ```

### Custom (Ruby) Facts

- First, the module's `lib/` directory must be created.

  ```bash
  mkdir -p lib/facter
  ```

- Programs in this directory will be synced to the node at the start of the Puppet run.
- When the node runs Facter, it will execute these as well to provide the custom facts for catalog compilation.
- Scripts must end in the `.rb` extension.
- The code that defines a custom fact is implemented by two calls:
  - A Ruby block starting with `Facter.add('fact-name')`.
  - Inside the Facter code block, a `setcode` block that returns the fact's value.
- __Example:__ Get the cluster name from the hostname without numbers at the end (i.e. webserver01 and webserver02 would return 'webserver').

  **/etc/puppetlabs/code/environments/test/modules/puppet/lib/facter/cluster_name.rb**

  ```ruby
  Facter.add('cluster_name') do
    setcode do
      Facter::Core::Execution.exec("/bin/hostname -s | /usr/bin/sed -e 's/\d//g'")
    end
  end
  ```

  - This will be available as `$facts['cluster_name']` for use in manifests and templates.
- It's best to run the smallest command possible to return a value, and then manipulate that value using native Ruby.
  - __Example:__ The same scenario using native Ruby.

    ```ruby
    require 'socket'
    Facter.add('cluster_name') do
      setcode do
        hostname = Socket.gethostname
        hostname.sub!(/\..*$/, '') # Remove everything after the first period.
        hostname.gsub(/[0-9]/, '') # Remove digits and return revised name.
      end
    end
    ```

- Plugins including facts are synced from the modules to the node (`pluginsync`) during Puppets initial configuration phase, prior to catalog compilation.
  - This makes the fact available for immediate use in manifests and templates.


### Debugging

### Understanding Implementation Issues

## Determining Functions

### Puppet Functions

### Ruby Functions

### Using Custom Functions

## Creating Puppet Types

### Defining Ensurable

### Accepting Params and Properties

### Validating Input Values

### Defining Implicit Dependencies

### Learning More About Puppet Types

## Adding new Providers

### Determining Provider Suitability

### Assigning a Default Provider

### Defining Commands for use

### Ensuring the Resource State

### Adjusting Properties

### Taking Advantage of Caching

### Learning More About Puppet Providers

## Identifying New Features

## Binding Data Providers in Modules

### Using Data from a Function

### Using Data from Hiera

### Performing Lookup Queries

## Requirements for Module Plugins