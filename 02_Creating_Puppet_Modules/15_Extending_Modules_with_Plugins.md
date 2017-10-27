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

- For each property with a value that needs to be compared to the resource, you will need a getter and setter method.

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
    commands :sed => 'sed'

    def color # Getter
      sed('-e', 's/^color = \(.*\)$/\1/', filename)
    end

    def color=(value) # Setter
      sed('-i', '-e', 's/^color = /color = #{value}/', filename)
    end

    def motto # Getter
      sed('-e', 's/^motto = \(.*\)$/\1/', filename)
    end

    def motto=(value) # Setter
      sed('-i', '-e', 's/^motto = /motto = #{value}/', filename)
    end

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

    def flush
      echo("color = #{resource['color']}", '>', filename)
      echo("motto = #{resource['motto']}", '>>', filename)
      @property_hash = resource.to_hash
    end
  end
  ```

  - The `color` method retireves the current value for the property 'color', and the `color=(value)` method sets the color on the node.
  - The same applies for the 'motto' property.
- If there are many attributes, you may want to cache up the changes and write them out at once.
  - After calling setter methods, the resource will call the `flush` method, if defined.
  - In this case, it is faster to simply write a new file.
- The final call, `@property_hash = resource.to_hash`, caches the current values of the resource into an instance variable.
  - This can be used with caching.

### Providing a List of Instances

- If reading resources is low impact, you can use an `instances` class method to load all instances into memory.
  - This can improve performance compared to loading each one iteratively.
- Resource providers use this data when making commands like:

  ```bash
  puppet resource elephant
  ```

- To disable preloading, define it with an empty array.

  ```ruby
  self.instances
    []
  end
  ```

- To preload data, we need to construct the method to output each file in the `/tmp/elephants` directory and create a new object with the values.

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
    commands :sed => 'sed'
    commands :cat => 'cat'

    def color # Getter
      sed('-e', 's/^color = \(.*\)$/\1/', filename)
    end

    def color=(value) # Setter
      sed('-i', '-e', 's/^color = /color = #{value}/', filename)
    end

    def motto # Getter
      sed('-e', 's/^motto = \(.*\)$/\1/', filename)
    end

    def motto=(value) # Setter
      sed('-i', '-e', 's/^motto = /motto = #{value}/', filename)
    end

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

    def flush
      echo("color = #{resource['color']}", '>', filename)
      echo("motto = #{resource['motto']}", '>>', filename)
      @property_hash = resource.to_hash
    end

    self.instances
      elephants = ls('/tmp/elephants/')

      elephants.split("\n").collect do |elephant|
        attrs = Hash.new
        output = cat("/tmp/elephants/#{elephant}")
        output.split("\n").collect do |line|
          name, value = line.split(' = ', 2)
          attrs[name] = value
        end

        attrs[:name] = elephant
        attrs[:ensure] = present
        new(attrs)
      end
    end
  end
  ```

- The `self.instances` section reads each assignment in the elephant file, and assigns the value to the name in a hash.
  - Every instance of the resource is not available in the `@property_hash` variable.
  - `puppet agent` and `puppet apply` will load all instances from this metod, and match up resources in the database to the provider that returned their values.

### Taking Advantage of Caching

- If all instances are cached in memory, you don't need to read from disk every time.
  - The `exists?` method can be rewritten to simply:

    ```ruby
    def exists?
      @property_hash[:ensure] == 'present'
    end
    ```

- However, all resources that are created or deleted must be updated in memory as well.
  - Thus, the `create` and `delete` methods must be updated.

    ```ruby
    def create
      echo("color = #{resource['color']}", '>', filename)
      echo("motto = #{resource['motto']}", '>>', filename)
      @property_hash[:ensure] = 'present'
    end

    # ...

    def destroy
      rm(filename)
      @property_hash[:ensure] = 'absent'
    end
    ```

- Finally, the setter and getter methods can be updated.

    ```ruby
    def color
      @property_hash[:color] || :absent
    end

    def color=(value)
      @property_hash[:color] = value
    end
    ```

- Now that changes are being saved back to the hash, you must define the `flush` method to write changes back to disk.

### Learning More About Puppet Providers

## Identifying New Features

- __Features:__ Ruby classes that determine if a specific feature is available on the target node.
  - For example, create an elephant feature that is activated if the node has elephants installed.
  - Always written in Ruby.
  - Stored in the `<module>/lib/puppet/feature` directory.

  **/etc/puppetlabs/code/environments/test/modules/puppet/lib/puppet/feature/elephant.rb**

  ```ruby
  require 'puppet/util/feature'

  Puppet.features.add(:elephant) do
    Dir.exist?('/tmp') and
    Dir.exist?('/tmp/elephants') and
    !Dir.glob?('/tmp/elephants/*').empty?
  end
  ```

## Binding Data Providers in Modules

- The `lookup()` function and automatic parameter lookup in classes use the following sources for data:
  - The global data provider (Hiera) in `${codedir}/hiera.yaml`.
  - The environment data provider in `environment.conf`.
  - The module data provider in `metadata.json`.

- There are two steps to providing a data source specific to a module:
  1. Define `data_provider` in the module's `metadata.json` file.
  1. Create a function or Hiera config file as the data source for the module.

### Using Data from a Function

- Create a Ruby function named `data` in your module's namespace.
- The function can be in Puppet language, stored in `<module>/functions/data.pp`, or a Ruby function defined in `<module>/lib/puppet/functions/<module>/data.rb`.
  - __Example:__ Ruby function.

    ```ruby
    Puppet::Functions.create_function(:'mymodule::data') do
      def data()
        # The following is an example of code to be replaced.
        # Return a hash with parameter name to value mapping for the user class.
        return {
          'specialapp::user::id'   => 'value for parameter $id',
          'specialapp::user::name' => 'value for parameter $name',
        }
      end
    end
    ```

- Regardless of language, the function must return a hash that contains keys within the class namespace, exactly as how keys must be defined in Hiera data.
- Make sure to modify the `metadata.json` file in the module's directory to indicate `"data_provider": "function"`.
  - The `mymodule::data` function is now the module data source for the 'mymodule' module.

### Using Data from Hiera

- Create a module data configuration file that defines the Hiera hierarchy.
  - Must be named `hiera.yaml`.
  - Must be version 4.

    ```yaml
    ---
    version: 4
    datadir: data # Path to data files.
    hierarchy:
      - name: "OS Family"
        backend: json
        path: "os/%{facts.os.family}"
      - name: "common"
        backend: yaml
    ```

- Create the directory listed in `datadir` and populate with the Hiera data files.
- Lastly, specify `"data_provider": "hiera"` in the module's `metadata.json` file.

### Performing Lookup Queries

- The lookup strategy of global, then environment, then module data providers, would be queried as follows if you used the `function` data source.

  ```puppet
  class mymodule(
    # Will check global hiera, then environment data provider, then call mymodule::data() to get all values.
    Integer $id,
    String $user,
  ){
    # ...
  }
  ```

- If you used the `hiera` data source for the module, then parameter values would be independently looked up in each data source.

  ```puppet
  class mymodule(
    # Will check global Hiera, then environment data provider, then module Hiera data.
    Integer $id,
    String $user,
  ){
    # ...
  }
  ```

## Requirements for Module Plugins

- External fact programs should be placed in the `facts.d/` directory and be executable by the `puppet` user.
- External fact data should be placed in the `facts.d/` directory and have file extensions of `.yaml`, `.json`, or `.txt`.
- Functions written in Puppet should be placed in the `functions/` directory.
- Ruby functions should be placed in `lib/puppet/functions/<modulename>/` and be named after the function.
- Ruby features should be placed in `lib/puppet/features/` and be named after the feature.
- Ruby functions or templates that call custom functions need to prefix the function name with `function_`.
- Ruby functions or templates that call custom functions need to pass all input parameters in a single array.