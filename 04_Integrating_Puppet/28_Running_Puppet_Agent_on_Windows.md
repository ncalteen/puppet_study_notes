# Running the Puppet Agent on Windows

- Puppet on Windows with Puppet Labs Supported modules provides the ability to:
  - Create, modify, and remove users and groups.
  - Install and configure applications.
  - Manage registry keys and values.
  - Download and execute PowerShell and cmd scripts.
  - Control icons on the user's desktop.
  - Build IIS sites and applications.
  - Install and manage SQL server.
- All of the previous Puppet concepts still apply, however there are some differences between Windows and Linux nodes, which will be covered in the rest of this section.

## Installing Puppet on Windows

- Download the Puppet agent for Windows.
  - During manual installation, you will be prompted to set the Puppet server FQDN.
  - This can be set during silent install as well.

  ```cmd
  msiexec /qn /norestart /i puppet-latest.msi PUPPET_MASTER_SERVER=puppet.example.com
  ```

## Configuring Puppet on Windows

- The Puppet configuration file can be found at `C:\ProgramData\PuppetLabs\puppet\etc\puppet.conf`.
  - `C:\ProgramData\Puppetlabs\` on Windows is logically equivalent to `/etc/puppetlabs/` on Linux.
  - Configuration files follow the same format, but paths with spaces must be quoted.

## Running Puppet Interactively

- The Puppet installer creates a Start Menu folder with shortcuts to Puppet documentation, as well as direct access to Puppet commands such as Facter and Puppet Agent.
  - These require elevated permissions to run.
- After installation, the `puppet` command is added to `PATH` by default.
- Running the "Start Command Prompt with Puppet" initializes a cmd prompt with Puppet installation directories in the `PATH`.
  - The environment will contain the necessary settings for using Ruby `gem` and related commands.
  - Start this prompt and run `set` to see the preconfigured environment variables.
    - FACTER_DIR
    - FACTER_env_windows_installdir
    - HIERA_DIR
    - MCOLLECTIVE_DIR
    - PL_BASEDIR
    - PUPPET_DIR
    - RUBYLIB
    - SSL_CERT_DIR
    - SSL_CERT_FILE

## Starting the Puppet Service

- Use the following to stop, start, and query the Puppet service on Windows.

  ```cmd
  sc stop puppet
  sc start puppet
  sc query puppet
  ```

- You can configure the service to start at boot or run only on demand.
  - The space after the equals sign is mandatory.

  ```cmd
  sc config puppet start= disabled
  sc config puppet start= auto
  sc config puppet start= demand
  ```

## Debugging Puppet Problems

- Run the Puppet service with debug output.
  - You will find Puppet debug logs in the Event Viewer.

  ```cmd
  sc stop puppet
  sc start puppet --debug --logdest eventlog
  ```

## Writing Manifests for Windows

- Very similar to writing manifests for Linux, with some exceptions.
  - __Semicolon Path Separator:__ Windows uses a semicolon for path separation instead of a colon.
  - __Case-Insensitive:__ Files, users, groups, and other attributes are case-insensitive on Windows, but case-sensitive within Puppet.
    - Windows administrators must be aware of this when writing Puppet manifests.
  - __Windows Service Name:__ Always use the Windows short name for a service, not the display name.
  - __Filesystem Paths:__ Puppet uses forward slashes for filesystem paths.
  - __Double Backslash in Double Quotes:__ Backslashes in double quotes need to have an additional backslash to indicate it was the intended character.
    - Alternatively, place the entire path in single-quotes.
  - __Line Endings:__ File sources or templates for Windows hosts must contain the CR/LF combination.
    - If you use your own resource providers, the `flat` filetype handles this automatically.

## Finding Windows-Specific Modules

- There is a combined package of popular Windows modules on Puppet Forge, `puppetlabs/windows`.
- Use the chocolatey package manager for Windows where possible, as it provides a method for upgrading packages in place (with the default Windows package manager does not support).