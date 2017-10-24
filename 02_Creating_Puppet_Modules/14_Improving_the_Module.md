# Improving the Module

- Validating input data to ensure it conforms to expectations.
- Providing data to modules using Hiera.
- Classes and subclasses within a module.
- Definitions taht can be reused for processing multiple items.
- Utilizing other modules within your own.
- Documenting modules.

## Validating Input with Data Types

- You can define the type when declaring the parameter.
  - Both declares the parameter and validates the input.

    ```puppet
    class puppet (
        String $version                  = 'latest',
        Enum['running,'stopped'] $status = 'running,
        Boolean $enabled                 = true,
    ) {
        # ...
    }
    ```

- A parameter without an explicit type defaults to `Any`.
- You should also declare types for lambda parameters.

  ```puppet
  split($facts['interfaces']).each |String $interface| {
      # ...
  }
  ```

### Valid Types

- Types are hierarchical, so parent daata types accept child data types.
  - Data
    - Scalar
      - Boolean
      - String
        - Enum
        - Pattern
      - Numeric
        - Float
        - Integer
      - Regexp
    - Undef
  - Collection
    - Array
    - Hash
  - Catalogentry
    - Resource
    - Class

- There are a few special types that can match multiple values.
  - __Variant:__ Matches any of a list of types.
  - __Optional:__ Matches a specific type, or no value.
  - __NotUndef:__ Matches any time except `Undef`.
  - __Tuple:__ An array with specific data types in each position.
  - __Struct:__ A hash with specific data types for the key-value pairs.
  - __Iterable:__ Matches any Type or Collection that can be iterated over.
  - __Iterator:__ Produces a single value of an Iterable type.
    - Used by chained iterator functions to act on elements individually instead of copying the entire Iterable type.
- Use Scalar for parameter values when they can be of multiple types.

### Validating Values

- The type system allows for validating not only parameter type, but also the data within structured data types.

  ```ruby
  Array[Float]                                      # An array of only Float values.
  Array[Numeric[-10, 10]]                           # Array of only integers between -10 and 10.
  Hash[String, Any, 2, 4]                           # A hash with two or four pairs of String keys with Any key values.
  Array[Variant[Integer[1, 5], Enum['up', 'down']]] # An array with 'up', 'down', or 1-5 values.
  ```

### Testing Values

- Use the `=~` operator to compare a value against a type.

  ```ruby
  $input_val =~ String
  $my_list =~ Iterable
  ```

- You can also determine if a type is in a Collection.

  ```ruby
  String in $array_of_values
  ```

- The `case` and `selector` expressions can be used to compare types.

  ```ruby
  case $input_val: {
      Integer: {
          # ...
      }
      String: {
          # ...
      }
      # ...
  }
  ```

- A type comparison will match for an exact type, as wel as parents of that type.

  ```ruby
  'text' =~ String
  'text' =~ Scalar
  'text' =~ Data
  'test' =~ Any
  ```

- You can use `assert_type()` to generate your own validation failure messages.

  ```ruby
  assert_type(String[12], $password) |$expected,$actual| {
      fail "Use a longer password. (Provided: ${actual})"
  }
  ```

### Comparing Strings with Regular Expressions

- Also uses the `=~` operator.
- __Example:__ Determine which files are manifests.

  ```ruby
  $manifests = ${filenames}.filter |$filename| {
      $filename =~ /\.pp$/
  }
  ```

- Supports Ruby core Regexp.
- You can match against multiple Strings and regexes with the Patter data type.

  ```ruby
  $placeholder_names = $people.filter |$name| {
      $name =~ Pattern['alice', '^(?i:j.*doe)']
  }
  ```

### Matching a Regular Expression

- You can also compare to determine if something is a regexp type.

  ```ruby
  /foo/ =~ Regexp         # true
  /foo/ =~ Regexp[/foo/]  # true
  /foo/ =~ Regexp[/foo$/] # false
  ```

### Revising the Module

  **/etc/pupptlabs/code/environments/test/modules/puppet/manifests/init.pp**

  ```puppet
  class puppet (
      String $server                    = 'puppet.example.com',
      String $version                   = 'latest',
      Enum['running','stopped'] $status = 'running',
      Boolean $enabled                  = true,
      String $common_loglevel           = 'warning',
      Optional[String] $agent_loglevel  = undef,
      Optional[String] $apply_loglevel  = undef,
  ) {
      # ...
  }
  ```

## Looking Up Input from Hiera

- Retrieve Hiera data with the `lookup()` function.

  ```puppet
  $version = lookup('puppet::version')
  ```

- Explicit `lookup()` calls for parameters aren't required.
  - As long as the parameters are listed within the module's namespace, Hiera will load them automatically.

    **/etc/puppetlabs/code/hieradata/hostname/client.yaml**

    ```yaml
    ---
    classes:
      - puppet
    puppet::version: 'latest'
    puppet::status: 'stopped'
    puppet::enabled: false
    ```

- Without any function calss, these values will be applied to the module as parameter input, overriding class defaults.
- You cannot define input parameters as hash keys under the module name.

### Using Array and Hash Merges

- By default, automatic parameter lookup uses the 'first' lookup strategy.
- Two ways to retrieve merged results of arrays and hashes:
  1. Define a default merge strategy with the `lookup_option` hash from the module data provider.
  1. Use the `merge` parameter of `lookup()` to override the default strategy.
- The following merge strategies are supported:
  - __first:__ Returns the first value in priority order.
  - __unique:__ A flattened array of all unique values found.
  - __hash:__ A hash containing the highest priority key and its values.
  - __deep:__ Every key and all values from every level.

### Understanding Lookup Merge

```bash
puppet lookup --merge unique [parameter]
```

- Is equivalent to:

```puppet
$myvar = lookup({name => '[parameter]', merge => 'unique'})
```

### Specifying Merge Strategy in Data

- The downside to specifying a merge strategy in the lookup function is that you can no longer take advantage of automatic parameter loading.
- You can instead specify a `lookup_options` property in your data directly.

  ```yaml
  lookup_options:
    users::userlist:
      merge: unique
  ```

- The module authors can set their own default merge strategy within module data.
  - This can be overridden by declaring `lookup_options` in global or environment data, which is evaluated at a higher priority.

### Replacing Direct Hiera Calls

- Direct `hiera()` queries use only the global lookup scope.
  - Use `lookup()` to make use of environment and module data providers.
- The `lookup()` function will accept an array of attribute names.
  - Each name will be looked up in order until a value is found.
  - Only the result for the first name found is returned, but it could contain merged values.

## Building Subclasses

- Each subclass is named within the scope of the parent class.
  - __Example:__ Using separate `puppet::agent` and `puppet::master` classes.
- Each subclass should be a separate manifest file in the `manifests/` directory.
  - Named `[subclass].pp`.

  **/etc/puppetlabs/code/environments/test/modules/puppet/manifests/init.pp**

  ```puppet
  class puppet (
      String $version  = 'latest',
      String $loglevel = 'warning',
  ) {

  }
  ```

  **/etc/puppetlabs/code/environments/test/modules/puppet/manifests/agent.pp**

  ```puppet
  class puppet::agent (
      Enum['running','stopped'] $status = 'running',
      Boolean $enabled,
  ) inherits puppet {
      # Add all resources previously defined in the puppet class.
  }
  ```

  **/etc/puppetlabs/code/environments/test/modules/puppet/manifests/server.pp**

  ```puppet
  class puppet::server ( ) {

  }
  ```

- Any time you would need an if/then block to handle different needs on different nodes, use a subclass for improved readability.

  **/etc/pupptlabs/code/hieradata/common.yaml**

  ```yaml
  classes:
    - puppet::agent
  puppet::common_loglevel: 'info'
  puppet::version: 'latest'
  puppet::agent::status: 'stopped'
  puppet::agent::enabled: false
  ```

## Creating New Resource Types

- Puppet classes are singleton; only one copy of a class can exist in memory.
- Defined resource types are implemented by manifests and look similar to subclasses.
  - Placed in the `manifests/` directory.
  - Named within the module namespace.
  - File name should be `[typename].pp`.
  - Begin with parenthesis which define accepted parameters.
- Defined resource types are NOT singleton.

## Understanding Variable Scope

- Modules may only declare variables in their own namespace.
- Each subclass has its own scope.

  ```puppet
  class puppet::agent {
      # These two are the same and will cause errors.
      $status = 'running'
      $::puppet::agent::status = 'running'
  }
  ```

- A module may not create variables within tope scope, or another module's scope.
- Variables can be prefaced with an underscore to indicate they should not be accessed externally.
  - Currently this is not enforced, but will be in future versions.

### Using Out-of-Scope Variables

- You can access, but not change, variables in other scopes.

  ```puppet
  notify($variable)                 # Current, parent, node, or top scope.
  notify($::variable)               # Top scope.
  notify($::other_module::variable) # Module scope.
  ```

- Always use the most specific scope possible.

### Understanding Top Scope

- Facts submitted by the module.
- Variables declared in manifests outside of a module.
- Variables declared in the parameters block of an ENC's result.
- Variables declared at top scope in Hiera.
- Variables set byb the agent or master.
- Always use client-supplied facts from the `facts[]` hash.
- When using a Puppet server, enable `trusted_server_facts` and use the server-validated facts available in `server_facts[]` and `trusted[]`.
  - By following these rules you can assume any top-level variable was set by Hiera or an ENC's result.
- Avoid defining top-scope variables.
  - Declare all variables in a module or role namespace.

### Understanding Node Scope

- Looks like top-scope variables, but are specific to a node assignment.
- Its possible to declare variables within the node block, which would override top scope variables (if you are using them without the `$::` prefix).
- Its best practice to avoid node blocks and use Hiera data to assign classes.
- Its best to avoid using these in general cases.

### Understanding Parent Scope

- The scope of the class that the current class inherits.

  ```puppet
  class puppet::agent() inherits puppet {
      # puppet is the parent scope
  }
  ```

### Tracking Resource Defaults Scope

- It is possible to declare attribute defaults for a resource type.
- To change those defaults within a class, define it within the class scope.
  - To change them in every class, define them in a defaults class and inherit it in every class in the module.
- It is not uncommon to place module defaults in a params class.

  ```puppet
  class puppet::params() {
      $attribute = 'default value'
      Package {
          install_options => '--enable-repo=epel' # Resource default.
      }
  }
  ```

- If a resource default is not declared in the class, it will use the default declared in the parent scope, node scope, or top scope.
  - If a class is declared within another class, the parent scope is the calling class.
  - If a class inherits another class, the parent scope is the inherited class.
  - The parent scope can change depending on which class declares this class first.
    - To counteract this, make sure each class inherits another base class.

### Avoiding Resource Default Bleed

- Puppet 4 provides the ability to implement named resource defaults which never 'bleed' into other modules.
  - A resource can accept multiple attributes from a hash containing attribute keys.
  - A resource declaration with a default resource body can provide default attributes.
  - By combining these two techniques, you can create resource defaults that can be applied by name.

  ```puppet
  $package_defaults = {
      'ensure'          => 'present',
      'install_options' => '--enable-repo=epel',
  }

  package {
      default:
        * => $package_defaults,
      ;
      'puppet-agent':
        ensure => 'latest', # Override resource default.
      ;
  }
  ```

- This works exactly as if a `Package{}` default was created, but applies only to the resources that specifically use the hash for defaults.

### Redefining Variables

- In previous versions, you could use `$sumtotal += 10` to declare a local variable based on computation of a variable in a parent, node, or top scope.
  - Removed in Puppet 4, as this was too similar to redefinition of the variable.
- Now, the below is used.

  ```puppet
  $sumtotal = $sumtotal + 10
  ```

- Generally, it is better to fully qualify higher-scope variables.

  ```puppet
  $sumtotal = $::sumtotal + 10         # Top scope
  # or
  $sumtotal = $::parent::sumtotal + 10 # Parent scope
  ```

- However, it is most recommended to not overlap variable names in different scopes.

## Calling Other Modules

- Previously, we created separate subclasses to configure the Puppet agent and server.
  - However, both read the `puppet.conf` config file, and subscribe to its changes.
- There are two ways to deal with this.

### Sourcing a Common Dependency

- Create a third class, `config`.
  - Contains a template to populate the config file with both agent and server settings.
  - Each class would include the config class.

  **/etc/puppetlabs/code/environments/test/modules/puppet/manifests/_config.pp**

  ```puppet
  class puppet::_config (
      Hash $common = {}, # Empty if not available in Hiera.
      Hash $agent  = {},
      Hash $user   = {},
      Hash $server = {},
  ) {
      file { 'puppet.conf':
        ensure  => file,
        path    => '/etc/puppetlabs/puppet/puppet.conf',
        owner   => 'root',
        group   => 'wheel',
        mode    => '0644',
        content => epp('puppet:///puppet/puppet.conf.epp', { 'agent' => $agent, 'server' => $server}),
      }
  }
  ```

- Note this does not require agent or server packages, nor does it notify the Puppet agent or server services.
  - These are different classes, and may not be declared on the same node.
  - It's only safe to depend on resources that are guaranteed to be in the catalog.
- We can now modify the agent class to include the _config class, and depend on its file resource.

  **/etc/puppetlabs/code/environments/test/modules/puppet/manifests/agent.pp**

  ```puppet
  class puppet::agent (
      String $status = 'running',
      Boolean $enabled,
  ) {
      include puppet::_config

      package { 'puppet-agent':
        version => $version,
        before  => File['puppet.conf'], # External class dependency.
        notify  => Service['puppet'],
      }

      service { 'puppet':
        ensure => $status,
        enable => $enabled,
        subscribe => [Package['puppet-agent'], File['puppet.conf']],
      }
  }
  ```

### Using a Different Module

- You can optionally use a separate module to make individual line or section changes to a file.
  - This is done without any knowledge of the remainder of the file.

  **/etc/pupptlabs/code/environments/test/modules/puppet/manifests/agent.pp**

  ```puppet
  class puppet::agent (
      String $status   = 'enabled',
      Boolean $running = true,
      Hash $config     = {},
  ) {
      # Write each config option to puppet.conf.
      $config.each |$setting,$value| {
          ini_setting { "agent $setting":
            ensure  => present,
            path    => '/etc/puppetlabs/puppet/puppet.conf',
            section => 'agent',
            setting => $setting,
            value   => $value,
            require => Package['puppet-agent'],
          }
      }
  }
  ```

## Ordering Dependencies

- Especially when working with separate modules maintained by other people, it is important to ensure dependencies are fulfilled before resources that depend on them are evaluated.

### Depending on Entire Classes

- When trying to set a dependency on a resource in another class, you must know the name of the resource.
  - This is prone to breakage if an update to the class renames the resource.
- Best practice is to treat the other class as a black box, and depend on it entirely.

  ```puppet
  service { 'puppet':
    # ...
    after => Class['rsyslog'],
  }
  ```

### Placing Dependencies Within Optional Classes

- The resource you notify or subscribe must exist in the catalog.
- Especially important when creating modules that may or may not be included with other modules.
- It is best to ensure that optional resources subscribe to required resources.
  - Required resources should not notify optional resources, in case they don't exist.
  - Always place the dependency in the optional resource.

### Notifying Dependencies from Dynamic Resources

- You must know the names of resources.
- Especially important within modules that depend on resources that are dymnamically generated.
- use the dynamic resource's `require` and `notify` metaparameters instead of `before` and `subscribe` on any required resources.

### Solving Unknown Resource Dependencies

- Issues can occur when you are writing a module that depends on resources dynamically generated in different classes.
- If that class can be used without your dependent class, then it cannot send a notify event, as your resource may not exist in the catalog.
  - If that class belongs to a module from Puppet Forge, you would have to vendor it to modify the class in question.
- __Escrow Refresh Resource:__ A well-known static resource that will always be available to notify or subscribe to.
  - Each dynamic resource notifies the escrow refresh resource if it is converged.
  - On a refresh, the escrow resource does something trivial.
  - Other dynamic resources can subscribe to the escrow resource.

    ```puppet
    exec { 'puppet-config-has-changed':
      command     => '/bin/true',
      refreshonly => true,
    }
    ```

  - The escrow refresh resource must be defined in the class on which the optional resources depend.
    - This may need to be submitted to the maintainer of the dependency.

## Containing Classes

- While a class can include another class, they are both defined at the same level.
  - Ordering metaparameters determine which classes are processed in which order.
- You can set dependencies and ordering on one class against any other class.
- Rather than have modules set dependencies on each subclass of the module, declare each subclass as contained within the 'main' class.
  - Any class that references the main class need not be aware of contained subclasses.

    ```puppet
    class application (
        Hash[String] $globalvars = {},
    ) {
        contain application::package
        contain application::package
    }

## Creating Reusable Modules

- Modules should be usable and repeatable in multiple scenarios.

### Avoiding Fixed Values in Attribute Values

- Use variables for resource attribute values.
  - Set the values in a params class, Hiera, or another source.

### Ensuring Fixed Values for Resource Names

- Resource namevars may vary based on type, OS, platform, etc.
- Use static names for resources to which wrapper classes may need to refer to with ordering metaparameters.

### Defining Defaults in a Params Manifest

- Place all conditional statements around OS and similar data in a params class and refer to it for all default values.

  ```puppet
  class apache_httpd (
      # ...
      String $package_name = $apache_httpd::params::package_name,
  ) {
      # ...
  }
  ```

## Best Practices for Module Improvements

- Declare parameters with explicit types for data validation.
- Validate each parameter for expected values in a manifest.
- Place each subclass and defined type in its own manifest.
- Avoid using top or node scope variables.
- Use the `contain()` function to wrap subclasses for simple dependency management.
- Use variables instead of fixed values for resource attributes.
- Use static names for resources which other modules may depend on.
- Create escrow resources for wrapper and extension modules to subscribe to.