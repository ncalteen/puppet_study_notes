# Customizing Environments

- Environments provide isolation for modules and data.
  - Support multiple team environments with different needs but some shared code.
  - Distinguish evolutions of product development (stable, beta, staging, test, etc.).
  - Avoid conflicts for dependency modules that share names.
  - Isolate security-sensitive data.
  - Stage tests for bug fixes or feature changes.

## Understanding Environment Isolation

- An environment is not a namespace.
- Data either exists in the environment or it does not.
  - Two nodes requesting data from two environments is logically the same as two nodes talking to two different Puppet servers.
- A module that doesn't appear in `modulepath` of an environment does not exist for that environment.
  - Two modules with the same name can exist without collision in two environments.
- Data that is not loaded for an environment does not exist, and cannot be referenced within the environment.

## Enabling Directory Environments

- Environments are enabled by default for Puppet 4.
- Puppet expects a directory for each environment in the location specified by `$environmentpath`, defaulting to `/etc/puppetlabs/code/environments`.
  - Each directory within this path is an environment name.
- Each environment directory should contain the following:
  - __environment.conf:__ An optional file to customize the environment.
  - __hieradata/:__ Configuration data for the environment.
  - __manifests/:__ Manifest files outside of any modules.
    - Only used for setting resource defaults for the environment.
  - __modules/:__ Location for modules available to the environment.
- Any modules placed in `$basemodulepath` will be shared among all modules.
  - Defaults to `/etc/puppetlabs/code/modules`.
  - Useful for sharing well-tested, common modules.

## Assigning Environment to Nodes

- A client informs the Puppet sever which environment to use based on the configuration file setting, command-line input, or the default of "production".
- A node terminus or ENC can assign a different environment than the one requested by the node.
  - There are no per-environment settings for ENCs, as the ENC must be queried before the environment is chosen.

## Configuring an Environment

- The `environment.conf` file for each environment is parsed after the main configuration file.
  - Options defined within `puppet.conf` can be used within the environment configuration.
- __config_version:__ A script that can be run to provide a configuration version number.
  - If not provided, the current epoch time is used.
- __manifest:__ The manifest file or directory that will be read before catalogs are built for this environment.
  - Defaults to the global setting for `default_manifest`, which is the `manifests/` directory within the environment.
- __modulepath:__ A list of directories to load modules from.
  - Defaults to `modules/` in the environment, followed by `basemodulepath`.
- __environment_data_provider:__ The name of a data provider class, or `function` to indicate that a function will be used to provide the environemnt data.
  - By default, none is provided.
- __environment_timeout:__ The cache timeout.
  - Stable environments should use a value of `unlimited`, and use a trigger to refresh the Puppet environment cache when new code is deployed.

  ```bash
  puppet config print --environment [environment]
  ```

## Choosing a Manifest Path

- If the manifest path is set to a directory, every `.pp` file in that directory will be read (in alphabetical order).
  - Otherwise, a specific file can be refrenced.
- Independent manifests in `[environment]/manifests` should contain only resource defaults for the environment.

## Utilizing Hiera Hierarchies

- There is only one Hiera configuration file common to all environments.
- If you have multiple teams using environments for delegation of control, you will likely want to separate data as well.
- You can use the environment path within the Hiera hierarchy.
  - For example, after the data directory, include the `$::environment` variable to create a directory structure separated out by environment.

    ```yaml
    :yaml:
      :datadir: /etc/puppetlabs/code/environments/%{::environment}/hieradata
    :json:
      :datadir: /etc/puppetlabs/code/environments/%{::environment}/hieradata
    ```

## Binding Data Providers to Environments

- As discussed earlier, parameter values are automatically sourced from the following locations in order:
  1. The global data provider (Hiera)
  1. The environment data provider specified in `environment.conf`.
  1. The module data provider specified in `metadata.json`.
- A data source can be added to an environment.
  - Define `environment_data_provider` in the `environment.conf` file.
  - Create a function or Hiera configuration file as the data source.
- Setting data providers in environments can replace the use of `$::environment` in the global Hiera `:datadir:`.
- Each environment can select a custom data hierarchy while still using global shared data.

### Querying Data from a Function

- Create a function named `environment::data()`.
  - A Puppet language function defined in `${environmentpath}/${environment}/functions/data.pp`, or a Ruby function in `${environmentpath}/${environment}/lib/puppet/functions/environment/data.rb`.

    ```ruby
    Puppet::Functions.create_function(:'environment::data') do
      def data()
        # Return a hash with parameter name to value mapping for the user class.
        return {
          'specialapp::user::id'   => 'Default for parameter $id',
          'specialapp::user::name' => 'Default for parameter $name',
        }
      end
    end
    ```

- The same function can be used across different environments.
- Modify the `environment.conf` file to set the `environment_data_provider` value.

  ```ini
  environment_data_provider = 'function'
  ```

- The environment data function will be queried after the Hiera global source, but before the module data source.

### Querying Data from Hiera

- To enable Hiera data for your environment, modify `environment.conf` to set `environment_data_provider = hiera`.
- Create a `hiera.yaml` file in the same directory.
  - Use a v4 configuration format.

  ```yaml
  ---
  version: 4
  datadir: hieradata
  hierarchy:
    - name: "Hostnme"
      backend: json
      path: "hostname/%{trusted.hostname}"
    - name: "common"
      backend: yaml
      path: common
  ```

- This data provider will be called after the global Hiera data provider, but before the module's data provider.

## Strategizing How to Use Environments

### Promoting Change Through Layers

- Develop changes to a module on a feature branch.
- Merge the module's changes to *master* for testing.
- Tag a new branch or merge to an existing release branch to push the changes to production.

  |Environment|Branch Name|Description                                           |
  |-----------|-----------|------------------------------------------------------|
  |dev        |dev        |Develop and test changes here.                        |
  |test       |master     |Merge to master and test on production-like nodes.    |
  |production |master     |Push master to production nodes and/or Puppet servers.|

- A small variation can support many more engineers, as well as feature and hotfix branches working on the same module at the same time.
  - A new environment is created for each feature branch.
  - After testing is complete, the change is merged to *test* for testing against a production-like environment.
  - The feature branch is destroyed when the change is merged to master and pushed to production.

  |Environment |Branch Name |Description                                                    |
  |------------|------------|---------------------------------------------------------------|
  |feature_name|feature_name|A new branch for a single feature or bug fix.                  |
  |test        |test        |Merge to test for review, Q/A, automated tests, etc.           |
  |production  |master      |Merge to master and push to production. Destroy feature branch.|

### Solving One-Off Problems Using Environments

- When fixing a production problem, it is often necessary to use a one-off test environment.
- Go into `$codedir/environments/` and create a new environment named for the problem.
- Two strategies for working on the one-off problem:
  - Check out everything from the production environment into this environment.
  - Set `modulepath` to include the production environment's modules, then check out only the problematic module into this environment's `modules/` directory.

    ```ini
    environment_timeout = 0
    manifest = $environmentpath/production/manifests

    # Option 2
    modulepath = ./modules:$environmentpath/production/modules:$basemodulepath
    ```

- A "copy-everything" approach provides more isolation and may be needed in larger situations/bugs.
- Once the fix is merged, remove the environment directory.
- Don't forget to copy or symlink `hiera.yaml` and the Hiera data directory if you are using an environment data provider.

### Supporting Diverse Teams with Environments

- Put shared modules in a directory specified in `basemodulepath`, usually `/etc/puppetlabs/code/modules`.
- Put global Hiera data in the `/etc/puppetlabs/code/hieradata` directory.
- Put team modules in the `/etc/puppetlabs/code/environments/[team]/modules/` directory.
- Enable `hiera` as the environment data provider in `/etc/puppetlabs/code/environments/[team]/environment.conf`.
- Define a team-specfic Hiera hieararchy in `/etc/puppetlabs/code/environments/[team]/hiera.yaml`.
- Put team data in `/etc/puppetlabs/code/environments/[team]/hieradata/`.

## Managing Environments with r10k

- Environments don't enable a diverse structure of many modules in multiple stages of development.
- r10k provides a way to track and manage dependencies that fits easily into a branch-oriented workflow.
- Puppet Enterprise includes r10k in Code Manager.
- Create a cache directory for r10k.

  ```bash
  sudo mkdir /var/cache/r10k
  sudo chown vagrant /var/cache/r10k
  sudo getm install r10k
  ```

### Listing Modules in the Puppetfile

- Within each environment directory, the `Puppetfile` will list modules that will be included in your installation.

  ```ini
  # Puppet Forge
  forge "http://forge.puppetlabs.com"
  moduledir = 'modules'                # Relative path to environment.

  # Puppet Forge Modules
  mod "puppetlabs/inifile", "1.4.1"    # Get a specific version.
  mod "puppetlabs/stdlib"              # Get latest, don't update again after.
  mod "jorhett/mcollective", :latest   # Update to latest version every time.

  # Track master from GitHub.
  mod "puppet-systemsd",
      :git => "https://github.com/jorhett/puppet-systemsd.git"

  # Get a specific release from GitHub.
  mod "puppetlabs-strings",
      :git => "https://github.com/puppetlabs/puppetlabs-strings.git",
      # :branch => 'yard-dev'          # An alternate branch.

      # Define which version to install with one of the following.
      :ref => '0.2.0'                  # A specific version.
      # :tag => '0.1.1'                # A specific tag.
      # :commit => 'xxxxxxxxxx'        # A specific commit.
  ```

- Example: Development environments can track the master branch to get the latest module updates, whereas production environments can logk in to release versions.
- The Puppetfile can be validated via the CLI.

  ```bash
  cd /etc/puppetlabs/code/environments/[env-name]
  r10k puppetfile check
  ```

- r10k can also be used to remove modules not specified in the Puppetfile.
  - Useful after a major update, where some modules are no longer required.

    ```bash
    cd /etc/puppetlabs/code/environments/[env-name]
    r10k puppetfile purge -v
    ```

- To prevent r10k from removing a module you are building, you can add an entry to Puppetfile that tells r10k to ignore it.

  ```ini
  # Still in progress.
  mod 'my-test', :local => true
  ```

### Creating a Control Repository

- Contains a map of environments in use.
- Add the following from the production environment.
  - `environment.conf`
  - `Puppetfile`
  - `manifests/*`
  - `hieradata/*`
- Alternatively, clone the Puppet Labs control repository template.

### Configuring r10k Sources

- A configuration file must be created at `/etc/puppetlabs/r10k/r10.yaml`.

  ```yaml
  # The location to use for storing cached Git repos.
  cachedir: '/var/cache/r10k'

  # A list of repositories to create.
  sources:
    # Clone the Git repository and instantiate an environment per branch.
    example:
      basedir: '/etc/puppetlabs/code/environments'
      remote: 'git@github.com:jorhett/learning-mcollective'
      prefix: false

  # An optional command to be run after deployment.
  # postrun: ['/path/to/command', '--opt1', 'arg1', 'arg2']
  ```

- You can list multiple sources for teams with their own control repositories.
  - Enable the prefix option if the same branch name exists in multiple sources.
  - Almost every control repo has a production branch.

  ```yaml
  sources:
    storefront:
      basedir: '/etc/puppetlabs/code/environments'
      remote: 'git@github.com:example/storefront-control'
      prefix: true
    warehouse:
      basedir: '/etc/puppetlabs/code/environments'
      remote: 'git@github.com:example/warehouse-control'
      prefix: true
  ```

### Adding New Environments

- Create a branch of this repository for each environment in the `$codedir/environments` directory.
- Commit environment-specific changes to that particular branch.
- Each branch is automatically checked out into an environment path of the same name.

### Populating a New Installation

- Use the `display` command to see what r10k is setting up for you.

  ```bash
  $ r10k deploy display -v
  example (/etc/puppetlabs/code/environments)
    - test
        modules:
          - inifile
          - stdlib
          - mcollective
          - systemsd
          - strings
    - production
        modules:
          - inifile
          - stdlib
          - mcollective
          - systemsd
          - strings
  ```

- All existing Puppet server configuration can be quickly replaced with the `deploy` command.

  ```bash
  r10k deploy environment -p -v
  ```

### Updating a Single Environment

- To check out or update the files in an environment, use the below command.

  ```bash
  r10k deploy environment test -v
  ```

- To update the modules specified by an environments `Puppetfile`:

  ```bash
  r10k deploy environment test --puppetfile -v
  ```

- Update a specific module in one or more environments.

  ```bash
  r10k deploy module stdlib -v
  r10k deploy module -e test mcollective -v
  ```

### Replicating Hiera Data

- You may want to deploy Hiera data along with modules.
- Add `hiera.yaml` and `hieradata/` to the  control repository.
  - Common strategy when the same people edit Puppet code and data.
- An alternative would be to place Hiera data in its own repository and use r10k to install the data separately from the code.
  - Useful when you ahve different permissions or deployment processes for data changes and code changes.
  - Add the hiera repo as a source to the r10k configuration in `/etc/puppetlabs/r10k/r10k.yaml`.

    ```ini
    sources:
      control:
        basedir: '/etc/puppetlabs/code/environments'
        remote: 'git@github.com:example/controll
        prefix: false
      hiera:
        basedir: '/etc/puppetlabs/code/hieradata'
        remote: 'git@github.com:example/hieradtata
    ```

  - Add `data_provider = hiera` to `environment.conf`, or add the environment directory to the global dta provider with the `$::environment` variable in `/etc/puppetlabs/code/hiear.yaml`.
- Both the code and data repositories must use the same branch name, otherwise they will be added to separate environments.

## Invalidating the Environment Cache

- As mentioned previously, its recommended to configure stable environments with `environment_timeout` set to `infinite`.
  - Provides higher performance for remote clients.
  - Ensures updates wont propagate to nodes accidentally.
- After pushing changes to the environment, you will ask the Puppet server to invalidate the environment cache.
  - This is available as an API call to the Puppet server.
  - A client key is required for this, as it involves special privileges.

    ```bash
    # This will generate a new client key for the machine making invalidation requests.
    puppet config set server puppet.example.com
    puppet agent --certname code-deployment --test --noop

    # Sign the cert on the Puppet server.

    puppet config set server puppet.example.com
    puppet agent --certname code-deployment --no-daemonize --noop
    ```

  - Add rules to permit this certificate access to delete the environment cache (specified in `/etc/puppetlabs/puppetserver/conf.d/auth.conf`).
  - Below section must be added before the default deny rule.

    ```hocon
    {
      match-request: {
        path: "/puppet-admin-api/v1/environment-cache"
        type: path
        method: delete
      }
      allow: [ code-deployment ]
      sort-order: 200
      name: "environment-cache"
    }
    # Default deny rule
    ```

    ```bash
    # Restart the Puppet server to enforce the change.
    sudo systemctl restart puppetserver
    ```

- Once authorized, the cache can be invalidated via a `curl` command.
  - A `204 No Content` response is expected.

  ```bash
  curl -i --cert .puppetlabs/etc/puppet/ssl/certs/code-deployment.pem \
    --key .puppetlabs/etc/puppet/ssl/private_keys/code-deployment.pem \
    --cacert ./puppetlabs/etc/puppet/ssl/certs/ca.pem \
    -X DELETE \
    https://puppet.example.com:8140/puppet-admin-api/v1/environment-cache
  ```

## Restarting JRuby when Updating Plugins

- After deploying changes to module plugins, inform Puppet Server of the changes so that it can restart JRuby instances to update their plugin code.
  - This should be added immediately after invalidating the environment cache.
- Use the below to permit access for the code-deployment key to call the necessary update API.
  - Again, make sure to add before the default deny rule.

    ```hocon
    {
      match-request: {
        path: "/puppet-admin-api/v1/jruby-pool"
        type: path
        method: delete
      }
      allow: [ code-deployment ]
      sort-order: 200
      name: "jruby-pool"
    }
    # Default deny rule.
    ```

    ```bash
    # Restart the Puppet server to activate the change.
    sudo systemctl restart puppetserver
    ```

- The JRuby pool can be restarted with a similar `curl` request.

  ```bash
  curl -i --cert .puppetlabs/etc/puppet/ssl/certs/code-deployment.pem \
    --key .puppetlabs/etc/puppet/ssl/private_keys/code-deployment.pem \
    --cacert ./puppetlabs/etc/puppet/ssl/certs/ca.pem \
    -X DELETE \
    https://puppet.example.com:8140/puppet-admin-api/v1/jruby-pool
  ```