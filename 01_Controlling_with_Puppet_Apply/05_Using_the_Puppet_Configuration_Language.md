# Using the Puppet Configuration Language

## Defining Values

- Variables are named storage of data, and cannot be reassigned!
- Prefaced with '$'.
- Variables must start with a lowercase letter or underscore, and may only contain lowercase letters, numbers, and underscores.
  - Variables starting with underscores should only be used in local scope.
- Variables that have not been initialized will have an undefined value (`undef`).
- You can determine a variable's type with `typeof()`.

  ```puppet
  include stdlib
  $myname = 'John'
  $nametype = typeof($myname) # 'String'
  ```

### Defining Numbers

- Unquoted numerals are evaluated as a Numeric data type.
  - Decimals start with 1 through 9.
  - Floating-point numbers contain a period.
  - Octal numbers start with a 0.
  - Hexidecimal numbers start with 0x.
- Any time an unquoted word starts with a number, it will be evaluated as Numeric.

### Creating Arrays and Hashes

- Arrays can contain mixed value types, including other arrays.

  ```puppet
  $my_num = [1, 2, 3]
  $my_list = [1, 2, 'Steve']
  $list_of_list = [1, 2, ['hello', 4]]
  ```

- Array-to-array assignment works as long as both arrays have equal length.

  ```puppet
  [$var1, $var2, $var3] = [1, 2, 3]
  [$var1, $var2] = [1, 2, 3] # Will not work!
  ```

- Some functions require a list of input values, instead of an array.
  - The 'splat' operator, `*`, will convert an array into a comma-separated list of values.

  ```puppet
  myfunction(*$array_of_arguments){ ... }
  ```

### Mapping Hash Keys and Values

- Member values are associated with a key value.
- Hash keys may be scalar, but values can be any data type.

  ```puppet
  $user = {
      'username' => 'Jill',
      'uid'      => 1001,
      'create'   => true,
  }
  ```

### Using Variables in Strings

- Strings without variables should be surrounded by single quotes.

  ```puppet
  $mystring = `No variables here.`
  ```

- Use double-quotes when interpolating variables into strings.

  ```puppet
  notice("My name is ${name}")
  ```

- __Heredoc Notation__: Using start and end tags to specify a multi-line string value.

  ```puppet
  $message = @(END)
    This message
    is on two lines.
  END
  ```

- By default, heredoc notation is not interpolated, but can be if you place the ending tag in double-quotes.

  ```puppet
  $message = @("END")
    This is my
    ${variable}
  END
  ```

- Escape sequences can be used in interpolated strings.
  - \n -> Line Feed
  - \r -> Carriage Return (Required in Windows)
  - \s -> Space
  - \t -> Tab

### Using Braces to Limit Problems

- Braces shold be used to delineate variable boundaries.

  ```puppet
  notice("The second value in my list is ${mylist[1]})
  ```

- Don't use braces outside of string interpolation.

  ```puppet
  file { $filename:
    ensure  => present,
    mode    => '0644',
    replace => true,
    content => $greeting,
  }
  ```

### Preventing Interpolation

- Use a backslash before a character to treat it as a literal.

  ```puppet
  $with_quotes = 'This string has \"double quotes\" in it.'
  $describe = "\$user uses a \$ and a \\"
  ```

- Previously interpolated values won't be interpolated again.

  ```puppet
  $inform = "${describe}, resolved"
  # Outputs "$user uses a $ and a \, resolved"
  ```

### Using Unicode Characters

- Unicode characters can be assigned to strings using \u followed by:
  - The UTF-16 four-digit number, such as \u20AC.
  - The UTF-8 hex value in braces, such as \u{C2A5}.

### Avoiding Redefinition

- Variables may not be redefined within a given namespace or scope.
  - A manifest has a single namespace, and a variable cannot receive a new value in that namespace.
- In declarative language, the interpreter handles variable assignment independently of usage within resources.

  **~/double-assign.pp**
  ```puppet
  $myvar = 5
  $myvar = 10
  ```

  ```bash
  puppet apply double-assign.pp # Will throw an exception.
  ```

### Avoiding Reserved Words

- Variables may be assigned string values as "bare" words, without quotes:

  ```puppet
  $name = nick   # Will work.
  $name = 'nick' # Best practice.
  ```

- Reserved words have special meaning to the interpreter and must be quoted when used in string values.
  - and
  - attr
  - case
  - class
  - default
  - define
  - else
  - elsif
  - false
  - function
  - if
  - in
  - import
  - inherits
  - node
  - private
  - or
  - true
  - type
  - undef
  - unless

## Finding Facts

- Facts output by facter are available for reference in your manifests.

  ```bash
  facter
  facter | grep version
  ```

- Puppet adds its own facts for use in modules.

  ```bash
  facter --puppet
  ```

  - The following are added about the system facts provided by Facter.
    - `$facts['clientcert']`: The client-reported certname value.
    - `$facts['clientversion']`: The client-reported Puppet version on the node.
    - `$facts['clientnoop']`: Weather noop was enabled in the configuration or command line.
    - `$facts['agent_specified_environment']`: The environment requested by the client.
      - The Puppet server can override this.
      - If not provided, defaults to "production".
- Puppet can be used to also list facts in JSON format.

  ```bash
  puppet facts find
  ```

- You can also specify different output formats, if desired.

  ```bash
  facter --yaml
  puppet facts --render-as yaml
  # or
  facter --json
  puppet facts --render-as json
  ```

## Calling Functions in Manifests

- __Function__: Executable code that may accept input parameters, and may output a return value which can be used to provide a value to a variable.
  - Functions are run on the Puppet master if it exists.
  - Return values can be used in a variable assignment, or in string interpolation.

  ```puppet
  notify { 'md5_hash':
    message => md5($facts['fqdn']),
  }
  $ result = "The hash is ${md5($facts['fqdn'])}"
  ```

- Functions do not need to return a value, such as logging notifications.
  - `debug("message")`
  - `info("message")`

- Functions can be written in prefix-format, or Ruby-style chained format.

  ```puppet
  notice('this)   # Prefix
  'this'.notice() # Chained
  ```

## Using Variables in Resources

- Each data type has a different method, and often different rules.
- Constant strings without variables should be surrounded by single quotes.
- Strings with variables to be interpolated must be in double-quotes.
- No other data type should be quoted.
- Arrays are zero-indexed, so the first value is at position 0.
- Two array indicies can be provided to return a range of values.

  ```puppet
  $first = $my_list[0]
  $sub_list = $my_list[3,5]
  ```

- Specific items ina  hash can be accessed with a key and/or sub-key(s).

  ```ruby
  $name = $my_hash['name']
  $room = $my_hash['office']['room']
  ```

- Braces are only needed when interpolating values in a string, not with the variable itself.
- Retrieve specific values from a hash by assigning to an array of variables named for the key(s) you would like to retrieve.

  ```ruby
  [$jack] = $homes          # Same as $jack = $homes['jack']
  [$username, $uid] = $user # $username = $user['username']
                            # $uid = $user['uid']
  ```

- Refer to facts explicitly with the `$facts[]` hash.
- You can set `strict_variables` in `/etc/puppetlabs/puppet/puppet.conf` to cause an exception to be thrown if a variable is used that was never declared.

## Defining Attributes with a Hash

- A hash of attributes can be passed to a resource definition with the splat operator.

  ```puppet
  $resource_attributes = {
      ensure    => present,
      'mode'    => '0644',
      'replace' => true,
  }
  file { '/tmp/test.txt':
    source => 'test.txt',
    *      => $resource_attributes,
  }
  ```

- This is essential for creating many resources with similar attributes.

## Declaring Multiple Resource Titles

- A single declaration can be used to create multiple resources.
  - Pass an array of titles with the same resource body.

    ```puppet
    file { ['/tmp/one.txt', '/tmp/two.txt']:
      ensure => present,
      owner  => 'vagrant,
    }
    ```

- This only works when the title is used as the parameter that makes the resource unique (the namevar).

## Declaring Multiple Resource Bodies

- A single resource declaration can have multiple resource bodies, separated by ';'.

  ```puppet
  file {
    'file_one.txt':
      ensure => present,
      owner  => 'vagrant',
    ;
    'file_two.txt':
      ensure => present,
      owner  => 'root',
  }
  ```

- If one resource body has a title of `default` (without quotes), it is not used to create a resource.
  - It will specify the default attributes for other resource bodies.
  - Default attributes can be overridden within each individually declared resource.

  ```puppet
  file {
    default:
      ensure => present,
      owner  => 'vagrant',
    ;
    'file_one': path => 'file_one.txt';
    'file_two': path => 'file_two.txt';
  }
  ```

### Adding to Arrays and Hashes

- Items can be added to arrays and hashes.
- A new value is returned, as the existing object is not changed.

  ```ruby
  $my_list = [1, 2, 3]
  $new_list = $my_list + [4, 5, 6] # [1, 2, 3, 4, 5, 6]

  $my_hash = { 'name' => 'Joe' }
  $new_hash = $my_hash + { 'uid' => 1001 } # { 'name' => 'Joe', 'uid' => 1001 }
  ```

- A single value can be added to an array with the `<<` operator.

  ```ruby
  $longer_list = $my_list << 33    # [1, 2, 3, 33]
  $longer_list2 = $my_list << [33] # [1, 2, 3, [33]]
  ```

### Removing from Arrays and Hashes

- Similar to adding, subtracting from an array or hash will return a new object with the value removed.

  ```ruby
  $my_list = [1, 2, 3]
  $new_list = $my_list - 2 # [1, 3]
  $new_list2 = $my_list - [1, 2] # [3]

  $user = { 'name' => 'Joe', 'uid' => 1001, 'gid' => 1001 }
  $new_user = $user - 'name' # { 'uid' => 1001, 'gid' => 1001 }
  $new_user2 = $user - ['uid', 'gid'] # { 'name' => 'Joe' }
  ```

## Using Comparison Operators

- Evaluates to true or false.
- Number comparisons are straightforward.

  ```ruby
  4 != 4 # true
  3 < 4  # true
  3 >= 7 # false
  ```

- String equality comparisons are case insensitive.

  ```ruby
  'coffee' == 'coffee' # true
  'Coffee' == 'coffee' # true
  ```

- Substring matches are case sensitive.

  ```ruby
  'tea' in 'coffee' # false
  'Fee' in 'coffee' # false - case sensitive
  'fee' in 'coffee' # true
  ```

- Array an hash comparisons match only with equal length and value(s).
- The `in` comparison looks for a value in an array, or a key in a hash.
- You can compare values to data types.

  ```ruby
  true =~ Boolean # true
  false =~ Boolean # true
  7 =~ Boolean # false
  ```

## Evaluating Conditional Expressions

### if/elsif/else Statements

  ```ruby
  if ($coffee != 'hot') {
      notify { 'dont drink coffee': }
  } elsif ($scotch == 'cold') {
      notify { 'drink scotch': }
  } else {
      notify { 'drink water': }
  }
  ```

### unless/else Statements

  ```ruby
  unless ($facts['kernel'] == 'Linux') {
      notify { 'you are likely on Windows/OSX': }
  } else {
      notify { 'you are on Linux': }
  }
  ```

- Using unless may be considered bad practice, but general guidance is to use whichever form is more readable.

### case Statements

- Great for performing numerous comparisons against explicit values, other variables, regular expressions, or function results.

  ```ruby
  case $my_drink {
      'wine':            { include state::california }
      $stumptown:        { include state::oregon }
      /(scotch|whisky)/: { include state::scotland }
      is_tea($drink):    { include state::england }
      default:           {}
  }
  ```

- Always include a default option, even if it does nothing.

### Selector Statements
- Similar to case statements, except they return a value instead of execute code.

  ```ruby
  $native_of = $my_drink ? {
      'wine'     => 'california',
      $stumptown => 'oregon',
      default    => 'unknown',
  }
  ```

- A value must be returned in an assignment operation, so a default must always be included.
- The splat operator can be used to expand an array of values into choices that can be used in `case` or `select` statements.

  ```ruby
  $redhat_based = [ 'RedHat', 'Fedora', 'CentOS', 'Scientific', 'Oracle', 'Amazon' ]
  $libdir = $facts['osfamily'] ? {
      *$redhat_based => '/usr/libexec/mcollective',
      # ...
  }
  ```

## Matching Regular Expressions

- The match operator (`=~`) requires a string value on the left, and a regex on the right.
- Regexes can be used in four locations:
  - Conditional Statements
  - `case` Statements
  - Selectors
  - Node Definitions (deprecated)

## Building Lambda Blocks

- A lambda is a block of code that allows parameters to be passed in.
  - Basically, functions without a name.
- Commonly used with iterator functions.

  ```ruby
  |$param1,$param2| {
      # Iteration code.
  }
  ```

- Lambdas have their own variable space.
  - Other variables at higher scopes are available to the lambda.

  **/vagrant/manifests/mountpoints.pp**

    ```puppet
    each($facts['partitions']) |$name,$device| {
        notice("${facts['hostname']} has device ${name} with size ${device['size'])}")
    }
    ```

    ```bash
    puppet apply /vagrant/manifests/mountpoints.pp
    ```

## Looping Through Iterations

- Evaluate many items in an array or hash of data using a single lambda.

### each()

- Invokes a lambda once for each entry in an array, or each key-value pair in a hash.
- No response is expected.

  ```puppet
  split($facts['interfaces'], ',').each |$interface| {
      # Do something with each interface
  }
  ```

- If you want a counter on an array, pass two parameters to the lambda.
  - The first will track the index, and the second will track the value.

    ```puppet
    split($facts['interfaces'], ',').each |$index,$interface| {
        # Do something.
    }
    ```

- When passing a hash, using one parameter will give you a [key,value] array, or two parameters will pass the key and value, respectively.

  ```puppet
  each($facts['system_uptime']) |$key,$value| {
      # ...
  }
  each($facts['system_uptime']) |$uptime| {
      # $uptime[0] => key
      # $uptime[1] => value
  }
  ```

- The `each()` function returns the result of the last operation performed.
- __reverse_each()__: Evaluates the array or hash in reverse order.
- __step(N)__: Evaluates each Nth entry after the first in an array or hash.

### filter()

- Returns a filtered subset of an array or hash containing only entries that were matched by the lambda.
  - The lambda evaluates each entry and returns a positive result if it matches.

- __Example:__ Filter all facts with "ipaddress6" as the beginning of the key.

  ```puppet
  $ips = $facts.filter |$key, $value| {
    $key =~ /^ipaddress6?_/
  }
  ```

  ```bash
  puppet apply /vagrant/manifests/ipaddresses.pp
  cat /vagrant/manifests/ipaddresses.pp
  ```

### map()

- Returns an array from the results of a lambda.
- Call map() on an array or hash, and a new array is returned with the results (the original is not modified).
- The lambda's final statement should return a value to add to the result array.
- __Example:__ Pass in an array of interface names, and return an array of IP addresses for those interfaces.

  ```puppet
  $ips = split($facts['interfaces'], ',').map |$interface| {
    $facts["ipaddress_${interface}"]
  }
  ```

### reduce()

- Processes an array or hash and returns a single value.
- Takes in two parameters:
  - An array or hash.
  - An initial seed value.
  - If the seed is not provided, it will use the first element in the hash/array.
- The lambda should be written to perform aggregation, additiona, or some other function to operation on many values and return a single one.
- __Example:__ Pass the hash of partitions to add together all of their sizes.

  **/vagrant/manifests/partitions.pp**

    ```puppet
    $total_disk_space = $facts['partitions'].reduce(0) |$total, $partition| {
      notice("partition ${partition[0]} is size ${partition[1]['size']}")
      total + $partition[1]['size_bytes']
    }
    notice("Total space is ${total_disk_space}")
    ```

### slice()

- Creates chunks of a specified size from an array or hash.
- The output can change, depending on how you invoke it.
  - If a single parameter is passed between the pipes, the value passed into the lambda will be an array containing the number of items specified by the slice size (in parenthesis).

    **/vagrant/manifests/slices.pp**

      ```puppet
      [1,2,3,4,5,6].slice(2) |$items$| {
        notice("\$item[0] = ${item[0]}, \$item[1] = ${item[1]}")
      }
      ```

      ```bash
      $ puppet apply /vagrant/manifests/slices.pp
      $item[0] = 1, $item[1] = 2
      $item[0] = 3, $item[1] = 4
      $item[0] = 5, $item[1] = 6
      ```

- If you provide the same number of parameters as the slice size, each variable will contain one entry from the slice.
- Hash entries are always passed in key-value pair entries.
- Most invocations of `slice()` produce no output, so it can be ignored.

### with()

- Invokes a lambda exactly one time, passing the variables provided.
- Most commonly used to isolate variables to a private scope, not available to the main scope's namespace.

  ```puppet
  with('nick', 'alteen', 'cse') |$first, $last, $title| {
    notice("I am ${first} ${last}, ${title} at AWS")
  }
  ```

### Capturing Extra Parameters

- Most functions only produce one or two parameters for input to the lambda.
- `slice()` and `with()` can send an arbitrary number of parameters.
- You can prefix the final parameter with the splat '*' operator to indicate it will accept all remaining input arguments.
  - Referred to as "captures-rest".
- The captures-rest data type will always be an array.

  **/vagrant/manifests/hostfile_lines.pp**

    ```puppet
    # example: 192.168.250.6 puppet.example.com p.example.com puppetserver
    $host = $hosts_line.split(' ')
    with($host*) |$ipaddr, $hostname, *$aliases| {
      notice("${hostname} has IP ${ipaddr} and aliases ${aliases}")
    }
    ```

    ```bash
    puppet apply /vagrant/manifests/hostfile_lines.pp
    ```