# Writing Manifests

- __Manifest__: A file containing Puppet configuration language that describes the resources and their desired state.
  - Uses resources to define a policy to enforce on a node.
  - A base component for configuration policy and Puppet modules.

## Implementing Resources

- Resources are the smallest block of Puppet code.
  - A single element you wish to evaluate, create, or remove.
- Puppet includes many base resource types:
  - Users
  - Groups
  - Files
  - Services
  - Host File Entries
  - Packages
  - Notify
- You can also create your own resources.
- A simple resource example:

  ```puppet
  notify { 'greeting':
    message => 'Hello, world!',
  }
  ```

- Manifests are text files with a .pp extension.

  ```bash
  cat /vagrant/manigests/helloworld.pp
  ```

## Applying a Manifest

- A single manifest can be applied with `puppet apply`.

  ```bash
  puppet apply /vagrant/manifests/helloworld.pp
  ```

1. A catalog is compiled from the manifest.
1. Order of evaluataion is determined using any dependency data.
1. The target resource is evaluated to determine if changes need to be applied.
1. The resource is created, modified, or removed.
1. Feedback about the catalog application is provided.

## Declaring Resources

- Resource type format is always the same.

  ```pre
  # type { 'title':
  #  attribute => value,
  #}
  ```

- Within a manifest or set of manifests being applied together (a catalog), a resource of a given type can only be declared once with a given title.

## Viewing Resources

- Puppet can show you existing resources written in Puppet lanage.
  - This makes it easy to generate code from existing resources.

  ```bash
  $ puppet resource mailalias postmaster
  mailalias { 'postmaster':
    ensure    => 'present',
    recipient => ['root'],
    target    => '/etc/aliases',
  }
  ```

- Uses the same code used by Puppet to compare and alter system state.
- Some resources output by `puppet resource` include read-only attributes that cannot be set in a manifest.

## Executing Programs

- Use the `exec` resource to execute programs as part of your manifest.

  ```puppet
  exec { 'echo-holy-cow':
    path      => '/bin',
    cwd       => '/tmp',
    command   => 'echo "Holy cow!" > testfile.txt',
    creates   => '/tmp/testfile.txt',
    returns   => [0],
    logoutput => on_failure,
  }
  ```

- The `creates` attribute tells Puppet not to execute the command when the file exists, making the resource idempotent.

  ```bash
  puppet apply /vagrant/manifests/tmp-testfile.pp
  puppet apply /vagrant/manifests/tmp-testfile.pp
  cat /tmp/testfile.txt
  ```

### Was that Idempotent

- It is best to avoid using the `exec` resource wherever possible.
  - You have to declare how to make the change, as well as weather or not to make the change.
- This example will prevent the command from executing more than once, because of the `creates` attribute, but it would not repair the file if the content was manually changed.

## Managing Files

- The previous example can be replaced with a declarative `file` resource.

  ```puppet
  file { '/tmp/testfile.txt':
    ensure  => present,
    mode    => '0644',
    replace => true,        # Setting replace as false will create the missing
    content => 'holy cow!', # file, but not replace one that has changed.
  }
  ```

  ```bash
  puppet apply /vagrant/manifests/file-testfile.pp
  ```

### Finding File Backups

- Every file changed by a file resource is backed up on the node in a directory specified by `$clientbucketdir`.
- You can backup a file to this directory manually.

  ```bash
  sudo puppet filebucket --local backup /tmp/testfile.txt
  ```

- A list of backed up files can be found with the CLI as well.

  ```bash
  sudo puppet filebucket --local list
  ```

### Restoring Files

- Use the hash associated with the backup to view its contents.

  ```bash
  sudo puppet filebucket --local get [hash]
  ```

- You can also compare two files (local and/or backup).

  ```bash
  sudo puppet filebucket --local diff [file/hash] [file/hash]
  ```

- You can restore a backup to its original location, or a new one.

  ```bash
  sudo puppet filebucket --local restore [path] [hash]
  ```

## Avoiding Imperative Manifests

- Imperative code tells the interpreter what to do, when to do it, and how to do it.
- Declartive code tells the interpreter what it wants the result to be.
- The exec resource is dangerous, because Puppet has now knowledge of what the executed command actually did (only that it was executed).
  - In the case of modified files, backups would need to be compared to determine actual changes.