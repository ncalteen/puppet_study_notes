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

  **/etc/puppetlabs/code/environments/test/modules/puppet/manifests/init.pp**

  ```puppet
  class puppet (
    # Common parameters across all classes
    String $version         = 'latest',
    String $common_loglevel = 'warning',
  ){
    notice("cluster_name: ${facts['cluster_name']}")
  }
  ```

  ```bash
  sudo puppet apply --environment test /etc/puppetlabs/code/environemnts/test/manifests/
  ```

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

- For more optimization, you could use the `hostname` fact instead.

  ```ruby
  Facter.add('cluster_name') do
    setcode do
      hostname = Facter.value(:hostanme).sub(/\..*$/, '')
    end
  end
  ```

- Plugins including facts are synced from the modules to the node (`pluginsync`) during Puppets initial configuration phase, prior to catalog compilation.
  - This makes the fact available for immediate use in manifests and templates.

#### Avoiding Delay

- Use the `timeout` property of `Facter.add()` to define how many seconds the code should take to complete.
  - This will cause the fact evaluation to move on without an error, but without a value for the fact.
  - It may be important to account for this in manifest code!

    ```ruby
    Facter.add('cluster_name', :sleep, :timeout => 5) do
      setcode do
        Facter::code::Execution.exec("/bin/hostname -s | /usr/bin/sec -e 's/\d//g'")
      end
    end
    ```

#### Confining Facts

- Use a `confine` statement to limit which nodes will attempt to execute a fact.
  - This lists another fact name and valid values.
  - Only nodes whose value for the listed fact is in the valid list of values will execute the custom fact code.
  - __Example:__ Only nodes with a specific domain should provide this fact.

    ```ruby
    Facter.add('cluster_name') do
      confine 'domain' => 'example.com' # domain is the fact and example.com is the acceptable value.
      setcode do
        Facter::code::Execution.exec("/bin/hostname -s | /usr/bin/sec -e 's/\d//g'")
      end
    end
    ```

  - Multiple `confine` statements can be used at once, and lists of values can be provided to single `confine` statements.
  - __Example:__ Restrict a fact to only Debian-based systems.

    ```ruby
    Facter.add('cluster_name') do
      confine 'operatingsystem' => %w{ Debian Ubuntu }
      setcode do
        Facter::code::Execution.exec("/bin/hostname -s | /usr/bin/sec -e 's/\d//g'")
      end
    end
    ```

#### Ordering by Precedence

- You can define multiple methods (resolutions), to get a fact value.
- Puppet will use the highest precedence resolution that returns a value.
- Useful when the source of a fact differs per OS.
- Facter discards any resolutions where the confine statements don't match.
- Facter tries each resolution in descending order of weight.
- When a value is returned, no further resolutions are tried.
- If no weight is defined, weight is equal to the number of confine statements in the block.
- __Example:__ Find hostname from two different configuration files.

  ```ruby
  Facter.add('configured_hostname') do
    has_weight 10 # Set the weight
    setcode do
      if File.exists? '/etc/hostname'
        File.open('/etc/hostname') do |fh|
          return fh.gets
        end
      end
    end
  end

  Facter.add('configured_hostname') do # Same fact name.
    confine "os['family']" => 'RedHat' # If has_weight was not set, this would have weight of 1.
    has_weight 5
    setcode do
      if File.exists? '/etc/sysconfig/network'
        File.open('/etc/sysconfig/network').each do |line|
          if line.match(/^HOSTNAME=(.*)$/)
            return line.match(/^HOSTNAME=(.*)$/)[0]
          end
        end
      end
    end
  end
  ```

#### Aggregating Results

- Defined differently than a normal fact.
  - `Facter.add()` must be invoked with a property `:type => :aggregate`.
  - Each data source is defined by a `chunk` code block.
  - The `chunks` object will contain all results.
  - An `aggregate` codeblock will evaluate chunks to provide the final fact value (not `setcode`).
  - __Example:__

    ```ruby
    Facter.add('fact_name', :type => :aggregate) do
      chunk('any-name-one') do
        # ...
      end

      chunk('any-name-two') do
        # ...
      end

      aggregate do |chunks|
        results = Array.new
        chunks.each_value do |partial|
          results.push(partial)
        end
        return results
      end
    end
    ```

### Debugging

- Use `puppet facts find` to see the values of custom facts.
  - Add the `--debug` flag to see where facts are being loaded from.

### Understanding Implementation Issues

- External fact are evaluated first, and thus cannot reference Facter or Ruby facts.
- Ruby facts are evaluated later and can use external facts.
- External executable facts are forked instead of executed in the same process.
  - This can have performance implications.

## Defining Functions

- Executed during catalog compilation.
- Written in Puppet or Ruby.

### Puppet Functions

- Each function should be written in a separate file and stored in the `functions/` directory.
- File name should be the function name followed by `.pp`.
- The function defined in the module must be qualified with the module's namespace.
  - A function placed in the environment's `manifests/` directory should be qualified wiht the `environment::` namespace.
- __Example:__ Write a function that takes many types of input and returns a Boolean value.

  **/etc/puppetlabs/code/environments/test/modules/puppet/functions/make_boolean.pp**

  ```puppet
  function puppet::make_boolean( Variant[String, Number, Boolean] $inputvalue ) {
    case $inputvalue {
      Undef: { false }
      Boolean: { $inputvalue }
      Numeric: {
        case $inputvalue {
          0: { false }
          default: { true }
        }
      }
      String: {
        case $inputvalue {
          /^(?i:off|false|no|n|'')$/: { false }
          default: { true }
        }
      }
    }
  }
  ```

### Ruby Functions

- Functions written in Puppet can only use Puppet language tools.
  - Ruby functions are much more powerful.
- Should be stored in the `lib/puppet/functions/<modulename>` directory of the module, and named after the function followed by the `.rb` extension.
- __Example:__ Create the same function using native Ruby.

  **/etc/puppetlabs/code/environments/test/modules/puppet/lib/puppet/functions/puppet/make_boolean_ruby.rb**

  ```ruby
  Puppet::Functions.create_function(:'puppet::make_boolean_ruby') do
    def make_boolean_ruby( value )
      if [true, false].include? value
        return value
      elsif value.nil?
        return false
      elsif value.is_a? Integer
        return value == 0 ? false : true
      elsif value.is_a? String
        case value
        when /^\s*(?i:false|no|n|off)\s*$/
          return false
        when ''
          return false
        when /^\s*0+\s*$/
          return false
        else
          return true
        end
      end
    end
  end
  ```

#### Calling Other Functions

- You can invoke a custom Puppet function from another custom Puppet function with the `call_function()` method.
  - Scans the scope of where it was called to find and load the other function.
  - All values after the function name are passed as parameters to the other function.

  ```ruby
  Puppet::Functions.create_function(:'mymodule::outer_function') do
    def outer_function(host_name)
      call_function('process_value', 'hostname', host_name)
    end
  end
  ```

#### Sending Back Errors

- Raise an error of type `Puppet::ParseError`.

  ```ruby
  Puppet::Functions.create_function(:'mymodule::outer_function') do
    def outer_function(fact_value)
      raise Puppet::ParseError, 'Fact not available!' if fact_value.nil?
      # Other things to do if above is false...
    end
  end
  ```

- It is a design decision if you want to let it gracefully fail and return another value, or if you want the full catalog to fail.

### Using Custom Functions

- You can use both Puppet and Ruby functions in your manifest code.

  **/etc/puppetlabs/code/environments/test/modules/puppet/manifests/agent.pp**

  ```puppet
  class puppet::agent (
    # Parameters specific to the agent subclass
    Enum['running','stopped'] $status = 'running',
    Boolean $enabled                  = true,
  ) inherits puppet {
    include puppet::config

    notice("Install the $version version of Puppet, ensure it's $status, and set boot time start to $enabled")

    package { 'puppet-agent':
      ensure => $version,
      before => File['puppet.conf'],
      notify => Service['puppet'],
    }

    service { 'puppet':
      ensure => $status,
      enable => puppet::make_boolean( $enabled ), # Try with make_boolean_ruby as well.
      subscribe => [ Package['puppet-agent'], File['puppet.conf'] ],
    }
  }
  ```

  ```bash
  sudo puppet apply --environment test /etc/puppetlabs/code/environments/test/manifests/
  ```

- In Puppet templates, you can call functions from the `scope` object using `scope.function_<function_name>(<variables>)` format.

  ```erb
  <%= scope.function_puppet::make_boolean([input-variable]) -%>
  ```

## Creating Puppet Types

- Creating in a Ruby class is more powerful Puppet types.
- Each should be stored in `lib/puppet/type/` and end with the `.rb` extension.
- Define the type with `Puppet::Type.newtype()`

  **/etc/puppetlabs/code/environments/test/modules/puppet/lib/puppet/type/elephant.rb**

  ```ruby
  Puppet::Type.newtype( :elephant ) do # Ruby symbol is the type name.
    @doc = %q{Manages elephants on the node
      @example
        elephant { 'horton':
          ensure => present,
          color  => 'grey',
          motto  => 'After all, a person is a person, no matter how small.',
        }
    }
  end
  ```

### Defining Ensurable

- Most types are `ensurable`.
  - We compare the current state of the type to determine what changes are needed.

  **/etc/puppetlabs/code/environments/test/modules/puppet/lib/puppet/type/elephant.rb**

  ```ruby
  Puppet::Type.newtype( :elephant ) do # Ruby symbol is the type name.
    @doc = %q{Manages elephants on the node
      @example
        elephant { 'horton':
          ensure => present,
          color  => 'grey',
          motto  => 'After all, a person is a person, no matter how small.',
        }
    }

    ensurable
  end
  ```

- The provider for the type is required to define three methods to create the resource, verify it exists, and destroy it.

### Accepting Params and Properties

- __params:__ Values used by the type byt not stored or verifiable on the resource's manifestation.
  - Every type must have a `namevar` parameter.
- __property:__ A value that we can retrieve from the resource and compare values.

  **/etc/puppetlabs/code/environments/test/modules/puppet/lib/puppet/type/elephant.rb**

  ```ruby
  Puppet::Type.newtype( :elephant ) do # Ruby symbol is the type name.
    @doc = %q{Manages elephants on the node
      @example
        elephant { 'horton':
          ensure => present,
          color  => 'grey',
          motto  => 'After all, a person is a person, no matter how small.',
        }
    }

    ensurable

    newparam( :name, :namevar => true ) do
      desc "The elephant's name"
    end

    newproperty( :color ) do
      desc "The elephant's color"
      defaulto 'grey'
    end

    newproperty( :motto ) do
      desc "The elephant's motto"
    end
  end
  ```

### Validating Input Values

- You can perform additional input valudation for each property or param provided to the type.
- __Example:__ Make sure the elephant's color is from a known list, and that the motto is an alphanumeric string with letters, spaces, single quotes, and periods, and that 'gray' for color is replaced with 'grey'.
  - The `newvalues()` method provides a good test when descriptive errors are not required.

  **/etc/puppetlabs/code/environments/test/modules/puppet/lib/puppet/type/elephant.rb**

  ```ruby
  Puppet::Type.newtype( :elephant ) do # Ruby symbol is the type name.
    @doc = %q{Manages elephants on the node
      @example
        elephant { 'horton':
          ensure => present,
          color  => 'grey',
          motto  => 'After all, a person is a person, no matter how small.',
        }
    }

    ensurable

    # Make sure color and motto are provided.
    validate do |value|
      if self[:motto].nil? and self[:color].nil?
        raise ArgumentError, "Both color and motto are required input."
      end
    end

    newparam( :name, :namevar => true ) do
      desc "The elephant's name"
    end

    newproperty( :color ) do
      desc "The elephant's color"
      defaulto 'grey'

      # Switch 'gray' with 'grey'.
      munge do |value|
        case value
        when 'gray'
          'grey'
        else
          super
        end
      end

      # Verify color is from list.
      validate do |value|
        unless ['grey','brown','red','white'].include? value
          raise ArgumentError, "No elephants are colored #{value}"
        end
      end
    end

    newproperty( :motto ) do
      desc "The elephant's motto"

      # Make sure motto is not empty, and has alphanumeric, spaces, single quotes, and periods.
      newvalues(/^[\w\s\'\.]$/)
    end
  end
  ```

### Defining Implicit Dependencies

- You can `autorequire` dependent resources for your type.
  - __Example:__ The elephant resources will exist in `/tmp/elephants`, thus the directories must exist as well.

  **/etc/puppetlabs/code/environments/test/modules/puppet/lib/puppet/type/elephant.rb**

  ```ruby
  Puppet::Type.newtype( :elephant ) do # Ruby symbol is the type name.
    @doc = %q{Manages elephants on the node
      @example
        elephant { 'horton':
          ensure => present,
          color  => 'grey',
          motto  => 'After all, a person is a person, no matter how small.',
        }
    }

    ensurable

    autorequire(:file) do
      '/tmp/elephants'
    end
    autorequire(:file) do
      '/tmp'
    end

    # Make sure color and motto are provided.
    validate do |value|
      if self[:motto].nil? and self[:color].nil?
        raise ArgumentError, "Both color and motto are required input."
      end
    end

    newparam( :name, :namevar => true ) do
      desc "The elephant's name"
    end

    newproperty( :color ) do
      desc "The elephant's color"
      defaulto 'grey'

      # Switch 'gray' with 'grey'.
      munge do |value|
        case value
        when 'gray'
          'grey'
        else
          super
        end
      end

      # Verify color is from list.
      validate do |value|
        unless ['grey','brown','red','white'].include? value
          raise ArgumentError, "No elephants are colored #{value}"
        end
      end
    end

    newproperty( :motto ) do
      desc "The elephant's motto"

      # Make sure motto is not empty, and has alphanumeric, spaces, single quotes, and periods.
      newvalues(/^[\w\s\'\.]$/)
    end
  end
  ```

- You can also use `autobefore`, `autonotify`, and `autosubscribe` to create soft dependencies and refresh events in the same manner as the ordering parameters.

### Learning More About Puppet Types

- Running Puppet with the `--debug` option shows the loading of custom types and selection of the provider.

## Adding new Providers

- __Providers:__ Ruby classes that do the detailed work of evaluating a resource.
  - Handles all OS-specific dependencies.
  - __Example:__ There are more than 20 different providers for `package` resources due to the different package managers in use.
- Always written in Ruby.
- Stored in the `lib/puppet/provider/<type-name>` directory with the `.rb` extension.
- Define the provider with the `Puppet::Type.type()` method, followed by the `provide()` method with three inputs:
  - The provider name as a Ruby symbol.
  - Optional `:parent` that identifies a provider from which to inherit methods.
  - Optional `:source` that identifies a different provider that manages the same resources.

  **/etc/puppetlabs/code/environments/test/modules/puppet/lib/puppet/provider/elephant/posix.rb**

  ```ruby
  Puppet::Type.type( :elephant ).provide( :posix ) do
    desc "Manages elephants on POSIX-compliant nodes."
  end
  ```

  **/etc/puppetlabs/code/environments/test/modules/puppet/lib/puppet/provider/elephant/windows.rb**

  ```ruby
  Puppet::Type.type( :elephant ).provide( :windows ) do
    desc "Manages elephants on Windows nodes."
  end
  ```

### Determining Provider Suitability

- Each provider needs to define ways that Puppet can determine if it is suitable to manage the resource on a given node.

  **/etc/puppetlabs/code/environments/test/modules/puppet/lib/puppet/provider/elephant/posix.rb**

  ```ruby
  Puppet::Type.type( :elephant ).provide( :posix ) do
    desc "Manages elephants on POSIX-compliant nodes."

    confine :osfamily => ['redhat','debian','freebsd','solaris'] # Only *Nix OSes.
    confine :true => /^4/.match( clientversion ) # Only Puppet 4 clients.
    confine :exists => '/tmp' # Dir must exist.
    contine :feature => 'posix' # POSIX feature available.
  end
  ```

### Assigning a Default Provider

- Sometimes multiple providers can manage the same resource on a node.
  - The `yum` and `rpm` providers can on CentOS, for example.
- You can use `defaultfor` to declare a provider default for a given platform.

  **/etc/puppetlabs/code/environments/test/modules/puppet/lib/puppet/provider/elephant/posix.rb**

  ```ruby
  Puppet::Type.type( :elephant ).provide( :posix ) do
    desc "Manages elephants on POSIX-compliant nodes."

    defaultfor :osfamily => ['redhat','debian','freebsd','solaris']

    confine :osfamily => ['redhat','debian','freebsd','solaris'] # Only *Nix OSes.
    confine :true => /^4/.match( clientversion ) # Only Puppet 4 clients.
    confine :exists => '/tmp' # Dir must exist.
    contine :feature => 'posix' # POSIX feature available.
  end
  ```

### Defining Commands for Use

- The `commands` method lets you test for the existence of a file, and sets up an instance method you can use to call the program.
  - If the command cannot be found, then the provider is not suitable for this resource.

  ```ruby
  commands :echo => '/bin/echo'
  commands :ls => 'ls'
  ```

  - If `ls` and `echo` cannot be found on the node, this provider is not suitable.
  - Command invocation is placed in Puppet debug output.
  - Nonzero exit codes are trapped, and raised as Puppet::ExecutionFailure errors.

### Ensuring the Resource State

- The provider must validate the existence of the resource and determine what changes are necessary.

  **/etc/puppetlabs/code/environments/test/modules/puppet/lib/puppet/provider/elephant/posix.rb**

  ```ruby
  Puppet::Type.type( :elephant ).provide( :posix ) do
    desc "Manages elephants on POSIX-compliant nodes."

    defaultfor :osfamily => ['redhat','debian','freebsd','solaris']

    confine :osfamily => ['redhat','debian','freebsd','solaris'] # Only *Nix OSes.
    confine :true => /^4/.match( clientversion ) # Only Puppet 4 clients.
    confine :exists => '/tmp' # Dir must exist.
    contine :feature => 'posix' # POSIX feature available.

    filename = '/etc/elephants' + resource[name]

    # Needed commands
    commands :echo => 'echo'
    commands :ls => 'ls'
    commands :rm => 'rm'

    # Ensurable requires these methods.
    def create
      echo("color = #{resource['color']}", '>', filename)
      echo("motto = #{resource['motto']}", '>>', filename)
    end

    def exists?
      begin
        ls(filename)
      rescue Puppet::ExecutionFailure => e
        false
      end
    end

    def destroy
      rm(filename)
    end
  end
  ```

### Adjusting Properties

### Taking Advantage of Caching

### Learning More About Puppet Providers

## Identifying New Features

## Binding Data Providers in Modules

### Using Data from a Function

### Using Data from Hiera

### Performing Lookup Queries

## Requirements for Module Plugins