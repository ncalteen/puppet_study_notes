# Expressing Relationships

- Use metaparameters to create and manage relationships between resources.
- Relationships control the order in which resources are evaluated.
- Dependencies can be defined within or between manifests.
- Many-to-one relationships are supported.
- Puppet generates a dependency 'graph' during catalog compilation.

## Managing Dependencies

- By explicitly declaring dependencies, Puppet can enforce more of the catalog.
  - Should the entire catalog fail if one resource fails?
- If a dependency for a resource fails, neither it nor any resource that depends on it will be applied by Puppet.

## Referring to Resources

- Referred to as a 'resource reference'.

  ```puppet
  service { 'puppet':
    ensure  => running,
    enabled => true,
    require => Package['puppet-agent'],
  }
  ```

- Resources should be created with a lowercase type ('service'), but referred to using uppercase ('Package').

## Ordering Resources

- Use the `before` metaparameter to specify a resource should be applied __before__ the target resource.

  ```puppet
  package { 'puppet':
    ensure => present,
    before => Service['puppet'],
  }
  ```

- Use the `requires` metaparameter to specify a resource should be applied __after__ the target resource.

  ```puppet
  service { 'puppet':
    enabled  => true,
    requires => Package['puppet'],
  }
  ```

- Both are not required at the same time.

## Assuming Implicit Dependencies

- Many Puppet types define autorequire dependencies on other Puppet resources.

  ```puppet
  file { '/var/log':
    # ...
  }
  file { '/var/log/puppet':
    ensure      => directory,
    autorequire => File['/var/log'], # Implicit dependencies.
    autorequire => File['/var'],     # Automatically added by Puppet.
  }
  ```

- These are 'soft' dependencies.
  - They will only exist if the mentioned resource exists in the catalog.

## Triggering Refresh Events

- The `before` and `requires` metaparameters ensure dependencies are processed before the resources that require them.
- The `notify` and `subscribe` metaparameters are similar.
  - They send a refresh event to a dependent resource if a dependency has changed.
  - __Example:__ Restart the Puppet service after a new version of the agent is installed.

    ```puppet
    package { 'puppet-agent':
      ensure => latest,
      notify => Service['puppet'],
    }
    service { 'puppet':
      ensure    => running,
      enable    => true,
      subscribe => Package['puppet-agent'],
    }
    ```

- The refresh event has special usage with `exec` resources when the resource's `refreshonly` attribute is set to true.
  - The `exec` resource will not be applied except during refresh events.

## Chaining Resources with Arrows

- Related resources can be ordered using chaining arrows.

  ```puppet
  Package['puppet-agent'] -> Service['puppet']
  # Equivalent to a before/requires.
  ```

- You can also specify notify/subscribe relationships.

  ```puppet
  Package['puppet-agent'] ~> Service['puppet']
  ```

## Processing with Collectors

- A collector is a grouping of many resources together, which can be used to affect many resources at once.

  ```puppet
  User <||>                                               # All users in a manifest.
  User <|groups == 'wheel'|>                              # Users in a manifest with the 'wheel' group.
  Service <|(ensure == 'running') and (enabled == true)|> # Enabled services set to start at boot.
  ```

- With collectors, dependencies can be set for all resources of a given type.

  ```puppet
  Package <||> ~> Exec['update-facts']
  ```

## Understanding Puppet Ordering

- During catalog compilation, Puppet creates a dependency graph.
  - Uses Directed Acyclic Graph (DAG) model.
  - Ensures no loops in graph paths.
  - Implicit dependencies are added first.
  - Dependencies added via metaparameters are added second.
- Resources without dependencies are not guaranteed to be ordered in any way.
  - Consistency of ordering between runs is not guaranteed.
- Unrelated resources can use the `ordering` option to order their execution.
  - Accepts three values:
    - __title-hash:__ Orders unrelated resources randomly, but consistently between runs.
    - __manifest:__ Orders resources by declaration in the manifest(s).
    - __random:__ Orders resources randomly each run.
      - This is useful for identifying missing dependencies.

  ```bash
  puppet apply --ordering=random /vagrant/manifests/testmanifest.pp
  ```

## Debugging Dependency Cycles

- Dependency cycles can occur when two rsources depend on each other being created first.

  ```bash
  puppet apply /vagrant/manifests/depcycle.pp
  ```

- Puppet will let you output the DAG model of a compiled catalog.

  ```bash
  puppet apply /vagrant/manifests/depcycle.pp --graph
  # You will need to use an external tool to view the map.
  ```

### Avoiding the Root User Trap

- The dependency graph only tracks resources in the catalog.
- Avoid creating a root user in the catalog.
- Use the numeric `uid => 0` and `gid => 0` values to avoid creating the implicit root user dependency.

### Utilizing Stages

- Stage resources allow you to break up the Puppet run into multiple stages.

  ```puppet
  stage { 'initialize':
    before => Stage['main'],
  }
  stage { 'finalize':
    requires => Stage['main'],
  }
  ```

- You can then assign classes to stages with the `stage` metaparameter.
- Drawbacks:
  - Dependency cycles are hard to track between stages.
  - You cannot notify or subscribe across stages.
  - You cannot assign classes to stages with Hiera data.