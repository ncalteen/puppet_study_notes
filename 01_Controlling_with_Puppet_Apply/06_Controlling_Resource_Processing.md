# Controlling Resource Processing

- __Metaparameters:__ Common attributes that can be used with any resource, including both built-in and custom resource types.
  - Control how Puppet deals with the resource.

## Adding Aliases

- __namevar:__ An attribute that identifies the unique resource on a node.
  - If not set, it defaults to the title of the resource.
  - Uniquely identifies the resource manifestation.
- You can add an alias, or friendly name to a resource.
  - This is important if the resource's name/location on disk may change from one node to another.
- Specify a different value for the title than the `namevar` attribute.

  ```puppet
  file { 'testfile': # alias
    path => '/tmp/testfile.txt', # namevar
    # ...
  }
  ```

- Specify the alias directly with the alias metaparameter.

  ```puppet
  file { '/tmp/testfile.txt', # namevar
    alias => 'testfile',
    # ...
  }
  ```

## Preventing Action

- The `noop` attribute allows a resource to be evaluated, but prevents any changes from being applied during a convergence.
- __Example:__ Check if an updated puppet-agent package is available, but don't install it.

  ```puppet
  package { 'puppet-agent':
    ensure => latest,
    noop   => true,
  }
  ```

- The same can be accomplished by adding `--noop` on a manifest, allowing you to preview changes to be applied.

  ```bash
  puppet apply --noop /vagrant/manifests/tmp-testfile.pp
  ```

## Auditing Changes

- The `audit` attribute defines a list of attributes you want to track changes to.
- Changing any of these attributes will log a message during `puppet apply` or `puppet inspect` runs.
- This is useful if you do not wish to manage a file, but need to know if its content changes.

  ```puppet
  file { '/etc/hosts':
    audit => ['content', 'owner'],
  }

- Generally it is not used for resources managed by Puppet as these will log changes automatically.

## Defining Log Level

- The `loglevel` attribute allows you to identify the level at which changes to the resource should be logged.
- Similar to syslog levels.
  - debug
  - err
  - info
  - alert
  - notice
  - emerg
  - warning
  - crit

  ```puppet
  package { 'puppet-agent':
    ensure   => latest,
    loglevel => warning,
  }
  ```

## Filtering with Tags

- Used for selective enforcement of resources.
- Applying only some parts of a policy based on tags.
- Can be added to a resource as a single string, or an array of strings.

  ```puppet
  package { 'puppet-agent':
    ensure => present,
    tags   => ['package', 'puppet'],
  }
  ```

- Running `puppet apply` with the `--tags` flag will cause Puppet to apply the policy, affecting resources with the specified tag(s).

  ```bash
  puppet apply /vagrant/manifests/packagetag.pp --tags package
  ```

- Puppet automatically adds a tag to every resource, called "resource".

### Skipping Tags

- The `--skip_tags` flag can be used to selectively skip resources with certain tags.

  ```bash
  puppet apply /vagrant/manifests/packagetag.pp --skip_tags service
  ```

## Limiting to a Schedule

- The `schedule` metaparameter can limit when Puppet will make changes to a resource.
- You can define how many times (`repeat`) within a given hour, day, or week (`period`) a resource is applied to a node.
- You can also limit convergence to specific hours, days of the week, and more using the `range` and `weekday` attributes.

  ```puppet
  schedule { 'twice-daily':
    period => daily,
    repeat => 2,
  }

  schedule { 'business-hours':
    period  => hourly,
    repeat  => 1,                  # Once per hour
    range   => '08:00 - 17:00',
    weekday => ['Mon', 'Tue', 'Wed', 'Thu', 'Fri'],
  }
  ```

- `period` can be hourly, daily, weekly, monthly, or never.
- The `repeat` attribute limits how many times it can be applied in a period.
- The `range` attribute defines hours and minutes in a 24-hour period.
- The `weekday` attribute should be an array of short names, numbers (Monday = 1), or full English names.
- If the range crosses the midnight boundary and weekday is defined, then weekday applies to the start time, not the end time.
  - __Example:__ If `range` is '10:00 - 02:00' and `weekday` is ['Mon','Tue'], then the resource can be converged at 2:00 AM Wednesday, as this falls in the range for Tuesday.
- Add the `schedule` metaparameter to your resource, specifying the schedule resource created.

  ```puppet
  package { 'puppet-agent':
    ensure   => latest,
    schedule => 'business-hours',
  }
  ```

- This only affects when resources can be applied.
  - Resources will not be automatically applied; something still needs to call Puppet on the node.

### Utilizing Periodmatch

- The `periodmatch` attribute can be added to the schedule resoruce.
- Takes two values:
  - __distance (default): Prevents application of the resource until the period time has passed.
    - Hourly, daily, weekly, monthly.
    - If the `repeat` attribute is specified, the period/repeat distance is used.
      - I.E. 4 times per day.
  - __number:__ Prevents application of the same resource in the same period by the number of the period.
- When `periodmatch` is set to number, the repeat value cannot be greater than 1.

  ```puppet
  schedule { 'once-an-hour':
    period      => hourly,     # The resource can only be applied once per hour,
    periodmatch => distance,   # where the hour starts at the time it was last applied.
  }

  schedule { 'once-an-hour':
    period      => hourly,     # The resource can only be applied once per clock hour.
    periodmatch => number,     # I.E. Once at 1:57 and again at 2:01.
  }
  ```

  ```bash
  while true; do date; \
  > puppet apply /vagrant/manifests/schedule.pp; sleep 60; done
  ```

### Avoiding Dependency Failures

- A resource that is not applied due to a configured schedule will succeed for the purpose of dependency evaluation.
- Recommendations:

  1. Apply the same schedule to dependent resources.
  2. Use `onlyif` guards.
  3. Run Puppet with `--ignoreschedules`.

## Declaring Resource Defaults

- If a resource declaration of the same type does not explicitly declare an attribute, the value from the resource default will be used.

  ```puppet
  Package {
      schedule => 'after-working-hours',
  }

  package { 'httpd': } # Will inherit the default schedule.
  ```

- Declare defaults at the top of a manifest for readability.
- Resource defaults can bleed into other code that is declared within the manifest scope.
  - A 'cleaner' solution is to add default attributes to a hash, and apply them with the splat '*' operator.