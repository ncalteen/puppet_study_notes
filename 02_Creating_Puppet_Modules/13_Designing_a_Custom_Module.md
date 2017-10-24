# Designing a Custom Module

- The high-level module creation process is as follows.

  1. Name the module (properly).
  1. Generate an empty module skeleton.
  1. Write the base manifest.
  1. Identify files to be copied to the node.
  1. Create templates to customize files on the node.
  1. Test that the module works as expected.

## Choosing a Module Name

- Must begin with a letter and contain only lowercase letters, numbers, and underscores.
  - Hyphens are not allowed!
- Each module creates its own namespace, and only one module may exist in a Puppet catalog with a given name at the same time.
- It is possible to manage conflicting module names in separate environments.

### Avoiding Reserved Names

- __Reserved Names:__
  - main
  - facts
  - server_facts
  - settings
  - trusted

## Generating a Module Skeleton

- Use the `puppet module generate` command.

  ```bash
  $ puppet module generate [orgname]-puppet
  $ tree puppet
  puppet/
  ├── examples
  │   └── init.pp
  ├── Gemfile
  ├── manifests
  │   └── init.pp
  ├── metadata.json
  ├── Rakefile
  ├── README.md
  └── spec
      ├── classes
      │   └── init_spec.rb
      └── spec_helper.rb
  ```

### Modifying the Default Skeleton

- You may wish to add items to your default modules.
- A revised skeleton can be placed in `~/.puppetlabs/opt/puppet/cache/puppet-module/skeleton`.
- You can also install multiple skeletons in the directory of your choice, and then reference them when generating a module.

  ```bash
  puppet module --module_skeleton_dir=~/skels/rails-app generate [orgname]-railsapp
  ```

## Understanding Module Structure

- __manifests/:__ Directory where code manifests (classes) are read.
- __files/:__ Files served by your module.
- __templates/:__ Templates parsed for custom files.
- __lib/:__ Ruby facts or functions.
- __specs/:__ Unit tests to validate the manifests.
- __tests/:__ System tests to validate the manifests.
- __facts.d/:__ External facts to be distributed.
- __metadata.json:__ Version and module dependency information.

## Installing the Module

- Any new module must be added to an environment's `modulepath`.

  ```bash
  cd /etc/puppetlabs/code/environments/test/modules
  puppet module generate aws-puppet --skip-interview
  ```

- Add the new module's class to Hiera for classification on the client node.

  **/etc/puppetlabs/code/hieradata/hostname/client.yaml**

  ```yaml
  ---
  classes:
    - puppet
  ```

## Creating a Class Manifest

- Every module must contain an `init.pp` manifest, which must contain the module's definition for the base class.

  **/etc/puppetlabs/code/environments/test/modules/puppet/manifests/init.pp**

  ```puppet
  class puppet {

  }
  ```

### What is a Class

- Manifests use Puppet code to define configuration policy.
- Manifests contain resources that describe how their target type should be configured.
- Manifests execute immediately with the `puppet apply` command.
- A class is a manifest with special properties.
  - Resources are declared in the same manner as a manifest.
- A class is a manifest that can be called by name.
- A class has a namespace of the same name.
- A class is not used until called by name.
- A class may include, or be included by, other modules.
- A class may be passed parameters when called.

## Declaring Class Resources

  **/etc/puppetlabs/code/environments/test/modules/puppet/manifests/init.pp**

  ```puppet
  class puppet {
    package { 'puppet-agent':
      ensure => latest,
      notify => Service['puppet'],
    }
    service { 'puppet':
      ensure    => running,
      enable    => true,
      subscribe => Package['puppet-agent'],
    }
  }
  ```

- This is the same as if we declared the resources outside the class.

  ```bash
  puppet apply --environment test /etc/puppetlabs/code/environments/test/modules/puppet/manifests/
  ```

## Accepting Input

  **/etc/puppetlabs/code/environments/test/modules/puppet/manifests/init.pp**

  ```puppet
  class puppet (
      $version = 'latest',  # Default parameter - optional
      $status  = 'running',
      $enabled,             # No default - required
  ){
      # ...
  }
  ```

- This will error out if `$enabled` is not passed when calling the class, but can default to the provided values for `$version` and `$status`.
- __Puppet Lookup:__ Classes can receive parameter input from three possible places.
  1. Parameter values explicitly declared with the class.
  1. Parameters looked up in data providers (Hiera, environment, module).
      - The key will be searched for in Hiera (global), then environment and module data providers.
      - The first source with the key "wins", according to the merge configuration.
  1. Default values provided in the class definition.
      - Puppet's style guide states every class should include default values that cover the most common use case.

  ```bash
  puppet apply --environment test /etc/puppetlabs/code/environments/test/modules/puppet/manifests/
  # The first run will fail, as $enabled must be set.
  ```

## Sharing Files

- A module's files must reside in a `files/` directory within the module.
  - You can add subdirectories to better organize files.

    ```bash
    mkdir /etc/puppetlabs/code/environments/test/modules/puppet/files
    cd /etc/puppetlabs/code/environments/test/modules/puppet
    cp /etc/puppetlabs/puppet/puppet.conf ./files/
    ```

    **/etc/puppetlabs/code/environments/test/modules/puppet/files/puppet.conf**

    ```ini
    # ...
    [main]
      log_level = notice
    ```

- Add a file resource to the class manifest, specifying source instead of content.

  **/etc/puppetlabs/code/environments/test/modules/puppet/manifests/init.pp**

  ```puppet
  class puppet {
    # ...
    file { '/etc/puppetlabs/puppet/puppet.conf':
      ensure => file,
      owner  => 'root',
      group  => 'wheel',
      mode   => '0644',
      source => 'puppet:///modules/puppet/puppet.conf',
    }
  }
  ```

- Valid URLs for file sources are `puppet:///`, `file:///`, `http:///`, or `https:///`.
  - Since remote webservers don't provide checksums, set `checksum => 'mtime'` in the `file` resource declaration.
- Best practice is to store all files within the module.
- The `/modules/<module_name>` is a special path that maps to `files/` in the module.
- An array of sources can be provided, and Puppet will use the first file found in the list.

## Testing File Synchronization

  ```bash
  sudo puppet apply --environment test /etc/puppetlabs/code/environments/test/manifests
  ```

- You will want to manually revert the permissions of the `puppet.conf` file.

  ```bash
  sudo chown vagrant:vagrant /etc/puppetlabs/puppet/puppet.conf
  sudo chmod 0644 /etc/puppetlabs/puppet/puppet.conf
  ```

- Alternatively, you could restore from Puppet's filebucket.

  ```bash
  sudo puppet filebucket -l list | grep puppet.conf
  sudo puppet filebucket restore /etc/puppetlabs/puppet.conf [HASH]
  ```

## Synchronizing Directories

- The `file` resource has optional recurse and purge attributes for working with directories.

  ```puppet
  file { '/tmp/sync':
    ensure  => 'directory',
    recurse => true,        # Go into subdirectories.
    replace => true,        # Replace files that already exist.
    purge   => false,       # Don't remove files we don't have.
    links   => follow,      # Follow symlinks and modify the link target.
    # ...
  }
  ```

- Avoid syncing large (100MB+) files with Puppet.
  - Leverage a dedicated tool for this instead.

## Parsing Templates

- Templates must exist in `[module]/templates`.
- Set the template variables in your class module.

  **/etc/puppetlabs/code/environments/test/modules/puppet/manifests/init.pp**

  ```puppet
  class puppet (
      $version         = 'latest',
      $status          = 'running',
      $enabled         = true,
      $server          = 'puppet.example.com',
      $common_loglevel = 'warning',
      $agent_loglevel  = undef, # Only used if passed in when the class is declared.
      $apply_loglevel  = undef,
  ) {
      # ...
  }
  ```

- Puppet has two template parsers:
  - Puppet EPP templates which use Puppet variables and functions.
  - Ruby ERB templates which use Ruby language and functions.

### Common Syntax

- Tags encapsulate Ruby and Puppet code.

  ```epp
    <%=
      # Tag is replaced with the value of a variable or result of the code.
    %>

    <%
      # Nothing is returned unless the code prings output.
    %>

    <%#
      # Comments are removed from output.
    %>

    <%
      # Trailing newlines or whitespace are removed to prevent blank lines.
    -%>

    <%-
      # Leading whitespace and newlines are removed.
    %>

    <%=
      # Removes trailing whitespace after the result of the code or variable.
    -%>

    <%%
      # Double '%' symbols are replaced with single to prevent string interpolation.
    %%>
  ```

### Using Puppet EPP Templates

- In a `file` resource, replace the `source` attribute with a `content` attribute.
  - When the `epp()` function is called, it returns the actual content to write to the file.

    ```puppet
    epp('path/to/template.epp', $params_hash)
    ```

- The `epp()` function takes two arguments:
  - The URI of the EPP template.
    - Always `puppet:///<module>/<file>.pp`.
  - The hash of the input parameters.

    **/etc/pupptlabs/code/environments/test/modules/puppet/templates/puppet.conf.epp**

    ```erb
    [master]
      log_level = <%= $::puppet::common_loglevel %>
    [agent]
    <% if $puppet::agent_loglevel != undef { -%>
      log_level = <%= $::puppet::agent_loglevel %>
    <% } -%>
      server = <%= $::puppet::server %>
    [user]
    <% if $puppet::apply_loglevel != undef { -%>
      log_level = <%= $::puppet::apply_loglevel %>
    <% } -%>
    ```

    **/etc/puppetlabs/code/environments/test/modules/puppet/manifests/init.pp**

    ```puppet
    class puppet(
        # ...
    ) {
        # ...
        file { '/etc/pupptlabs/puppet/puppet.conf':
          ensure  => file,
          owner   => 'root',
          group   => 'wheel',
          mode    => '0644',
          content => epp('puppet:///puppet/puppet.conf.epp),
        }
    }
    ```

#### Providing Parameters

- Instead of specifying the fully-qualified name of the variable, you can pass a hash of variables into the `epp()` function.
  - In the file resource, specify a hash of inputs to the template.

    ```puppet
    content => epp('puppet:///puppet/puppet.conf.epp', {
        'server'          => $server,
        'common_loglevel' => $common_loglevel,
        'agent_loglevel'  => $agent_loglevel,
        'apply_loglevel'  => $apply_loglevel,
    })
    ```

- When providing parameter inputs, the first line of the template should specify the accepted inputs.

  ```erb
  <%- | String $server,
        String $common_loglevel,
        Optional['String'] $agent_loglevel = undef,
        Optional['String'] $apply_loglevel = undef,
  | -%>
  ```

- The input hash must match the template's input definition, or it will fail.

#### Iterating Over Values with EPP

- You can use any Puppet function within template tags.
- __Example:__ Use `reduce()` to iterate through an array of tags and their values.

  ```erb
  [agent]
    tags = <%= $taglist.reduce |$tags,$tagname| {
        sprintf("%s,%s", $tags, $tagname)
    } -%>
  ```

### Using Ruby ERB Templates

- Remove the `epp()` function, replacing with the `template()` function.
- Rename the file to use a `.erb` extension.

  ```puppet
  file { '/etc/puppetlabs/puppet/puppet.conf':
    ensure  => file,
    owner   => 'root',
    group   => 'wheel',
    mode    => '0644',
    content => template('puppet:///puppet/puppet.conf.erb'),
  }
  ```

- Each instance of `<%= @variable %>` is reaplced with the value of the Puppet variable.
- There is not an explicit list of variables for the template.
  - The variables in the template must exist in the same scope (within the module class) as the template definition.
- Call Puppet functions using `scope.function_<puppet_function>()`.

  ```erb
  server = <%= scope.function_fqdn_rand(3600) -%>
  ```

- You can put any Ruby statement within `<% ... %>` tags (without an equals sign).

#### Iterating Over Values with ERB

- Similar to the EPP example, use Ruby code to iterate through an array of tags.

  ```erb
  [agent]
    tags = <%= tags = ''; @taglist.each do |tagname|
        tags += tagname + ',' # There is no @ before the tagname variable, as
      end                     # we are not referencing a variable in the module class.
      tags.chop               # Remove trailing comma.
    %>
  ```

## Testing the Module

  ```bash
  sudo puppet apply --environment test /etc/puppetlabs/code/environments/test/manifests/
  ```

- Try setting enabled to true in Hiera to observe changes.

## Peeking Beneath the Hood

- __Don't use the techniques here for anything other than debugging!__
- Environments are strict.
  - You cannot see data or code that is not loaded in your active environment.
- Within an environment, calss boundaries are not enforced.
  - You can access both variables and resources within other classes.

    ```puppet
    class my_class ($idea = 'games') {
        $sentence = "an idea: ${idea}"
    }

    class other_class {
        notice("The idea: ${Class['my_class']['idea']}") # Class parameter.
        notice("The sentence: ${::my_class::sentence}")  # Class variable.
    }
    ```

    ```bash
    puppet apply /vagrant/manifests/peek.pp
    ```

## Best Practices for Module Design

- Create a module skeleton that implements your organizations standards for design.
- Declare a class with the module name in `manifests/init.pp`.
- Assign default parameter values that reflect the most likely use case.
- Use EPP templates with explicitly defined parameter values.
- Don't access external data within a template.
  - Assign external data to variables in the manifest.