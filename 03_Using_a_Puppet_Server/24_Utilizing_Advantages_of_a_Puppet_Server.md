# Utilizing Advantages of a Puppet Server

## Using Server Data in Your Manifests

- With Puppet Server, you gain several additional data sources not available with `puppet apply`.

### Trusted Facts

- When a node connects to a Puppet server, it transmits a list of facts about the node.
  - There is no way to validate if these facts are correct.
  - A compromised node could forge fact data that can be used to gain accass to secure information/configuration.
- The `$trusted[]` hash contins facts that have been validated by the server, and can be trusted to be correct.
- __$trusted['authenticated']:__ The possible values are:
  - __remote:__ Confirms a successful validation of the remote client's certificate.
  - __local:__ Indicates that `puppet apply` is being used and validation is unavailable.
  - __false:__ Warns that the `auth.conf` file is misconfigured to allow unauthenticated requests.
- __$trusted['certname']:__ This contains the certificate name validated by the Puppet server if remote, or the value of `$certname` in the Puppet configuration if local.
- __$trusted['hostname'] / $trusted['domain']:__ `hostname` will contain the part before the first period of the certificate name, and `domain` will contain the remainder.
  - This is useful when using FQDN cert names, but possibly confusing in other situations.
- __$trusted['extensions']:__ A hash containing all certificate extensions present in the certificate.
  - You can create additional extensions (to be discussed later).
- This can be used to increase the security of your Hiera configuration by setting the hierarchy to use trusted facts, instead of those provided by the client.

  ```yaml
  :hierarchy:
    - "nodes/%{truted.certname}"
    - "os/%{facts.osfamily}"
    - common
  ```

### Server Facts

- The server sets certain variables for use in manifests.
- If the node provides facts of the same name, the server values will be overridden.
- Enable the below setting for Puppet to define a new hash of server facts (that cannot be overridden).

  ```ini
  [master]
    trusted_server_facts = true
  ```

- This will create the `$server_facts[]` hash.
  - __$server_facts['serverversion']:__ Version of Puppet used by the server (not the version of Puppet Server).
  - __$server_facts['servername']:__ The certificate name of the server.
    - This is the name used to sign all certificates (if the server is the CA).
  - __$server_facts['serverip']:__ The IP address it was contacted on (can vary with distributed systems).
  - __$server_facts['environment']:__ The environment selected by the server for the node.
    - It is possible for the node to request a different environment.
    - This will always be set to the environment used to process the node's catalog.
- Always enable `trusted_server_facts`.
- Always refer to `facts[]` for client-provided facts.
- Always refer to `server_facts[]` for server-validated facts.

### Server Configuration Settings

- Puppet servers make every configuration setting available as `$::settings::[setting_name]`.

  ```puppet
  notice("Trusted server facts are enabled: ${::settings::trusted_server_facts}")
  ```

- There is no equivalent access to node configuration settings by default.
- To gain access to a node's configuration values, add a custom or external fact to a module.

## Backing Up Files Changed on Nodes

- Every file changed by a `file` resource, or reviewed by the `audit` rresource, is backed up in the `clientbucketdir` directory.
- You can move backups of file changes to the Puppet server instead.
  - Define a `filebucket` resource with the path attribute set to false.

    ```puppet
    filebucket { 'puppet.example.com':
      path   => false,
      server => 'puppet.example.com',
      port   => '8140',
    }
    ```

- As defaults are always the current server, you can back up every file to that server using the below:

  ```puppet
  filebucket { 'local_server':
    path => false,
  }
  ```

- This resource should be defined in the top-scope manifest, in the environment's `manifests/` directory.
- You can declare multiple filebucket resources, and declare within the resource, or in a resource default, which bucket to use.

  ```puppet
  file { '/etc/sudoers':
    # ...
    backup => 'security-files' # The name of the filebucket resource.
  }
  ```

- Backing up to the Puppet server is required for viewing file changes in the Puppet Dashboard.

## Processing Puppet Node Reports

- Don't waste time excessively tuning the log levels of every resource.
  - Node reports provide direct access to every detail of the convergence process.

### Enabling Transmission of Reports

- Agents communicating with a Puppet server send reports to the same server from which they receive the catalog.
  - This can be changed in `/etc/puppetlabs/puppet/puppet.conf`.

    ```ini
    [agent]
      report = true
      report_server = $server
      report_port = $masterport
    ```

- You should only add/change these values if you wish to disable the sending of reports or change the report processor.

### Running Audit Inspections

- If you have resources defined with the `audit` attribute, you can run `puppet inspect` to generate a node report of all audited resources.
  - This report will be sent to the Puppet server, or to a different server passed in the CLI.
- You can find the inspection report in the Puppet Dashboard, under the Node Details page.
- The `--archive-file_server` argument is optional and defaults to the configured Puppet server.
  - Backs up copies of audited files during inspection.
- You may want to run this from `cron` on all nodes.

  ```puppet
  cron { 'audit-file-changes':
    command => 'puppet inspect --archive_files',
    user    => 'root',
    hour    => '12',
    minute  => '0',
  }
  ```

### Storing Node Reports

- Reports are stored in YAML format in `reportdir`.
  - In the scenario we have configured, this would be `/var/opt/puppetlabs/puppetserver/reports`.
  - The default is `/opt/puppetlabs/puppet/cache/reports`.

    ```bash
    sudo -u puppet bash
    cd /var/opt/puppetlabs/puppetserver/reports
    ls -la
    ls -la client.example.com
    head -10 client.example.com/[report_id].yaml
    ```

- On the client node, you can find the same report under `/opt/puppetlabs/puppet/cache/state/last_run_report.yaml`.

### Logging Node Reports

- The Puppet agent will send each message at or above the configured log level to the configured log target (default is syslog).
- If your Puppet server has access to a centralized log server not available to agent nodes, then it can be helpful to have the server send all agent messages to its own logs.
- This can be enabled in `/etc/puppetlabs/puppet/puppet.conf`.

  ```ini
  [master]
    reports = log,store
  ```

### Transmitting Node Reports via HTTP

- Enabled transmission of node reports to a URL by adding the `http` action to the list of report procesors, and specifying a target with `reporturl` in `/etc/puppetlabs/puppet/puppet.conf`.

  ```ini
  [master]
    reports = http,store
    reporturl = https://puppet-dashboard.example.com/
  ```

- Data will be sent via POST with a Content-type of `application/x-yaml`.

### Transmitting Node Reports to PuppetDB

- Enable transmission of node reports to PuppetDB by adding the `puppetdb` action to the list of report processors.

  ```ini
  [master]
    storeconfigs = true
    storeconfigs_backend = puppetdb
    reports = puppetdb,store
  ```

### Emailing Node Reports

- Requires installation of the `puppetlabs-tagmail` module on the Puppet server.

  ```bash
  puppet module install puppetlabs-tagmail
  ```

- Create a `/etc/puppetlabs/puppet/tagmail.conf` file containing a map of Puppet tags and recipients.
  - The module will send an email containing specific tags to the configured recipient.
  - The tagmail report handler allows tags to be used for filtering changes to specific resources.
- Each line of `[tagmap]` should contain one of the below formats.
  - A list of tags or negated tags, followed by a colon and a comma-separated list of recipient emails.
  - A list of log levels or negated log levels, followed by a colon and a comma-separated list of recipient emails.

  ```ini
  [tagmail]
    # Log levels
    all: log-archive@example.com
    emerg: puppet-admins@example.com

    # Tags
    frontend, java: fe-dev-team@example.com, java-platform@example.com
    database, mysql: dba@example.com

  [transport]
    reportfrom = puppetserver@example.com
    sendmail = /usr/sbin/sendmail

    # Direct SMTP delivery
    # smtpserver = smtp-relay.example.com
    # smtphelo = puppet.example.com
    # smtpport = 25
  ```

- Enable transmission of email reports by adding the `tagmail` option to `puppet.conf`.

  ```ini
  [master]
    reports = store,tagmail
  ```

### Creating a Custom Report Processor

- Must be written in Ruby and follow the below rules:
  - You can write a wrapper in Ruby that calls another program.
  - The script must be named according to the rules for Ruby symbols (alphanumeric only), with a trailing `.rb`.
  - It must have a `require 'puppet'` at the top of the script.
  - It must call `Puppet::Reports.register_report()` with a block of code, pasing its own name as a symbol as the only parameter.
  - The code must contain:
    - A call to `desc` with a text or Markdown-formatted string that describes the report processor.
    - A defined method called `process` that does the actual processing.
- The `process` method will be an object of type `Puppet::Transaction::Report`.
  - It can retrieve details of the report by querying atttributes of `Formats::Reports`, or get YAML data by calling `self.to_yaml()`.
- __Example:__ Report processor that creates a different log message for each Puppet agent run.

  ```ruby
  require 'puppet'
  require 'syslog/logger'
  log = Syslog::Logger.new 'logActionTime'

  Puppet::Reports.register_report( :logActionTime ) do
    desc "Sends one log message containing agent action times"

    def process
      source = "#{self.time} #{self.host} "
      action = "did #{self.kind} with result of #{self.status} "
      version = "for config #{self.configuration_version} "
      environment = "in environment #{self.environment}"
      log.info source + action + version + envronment
    end
  end
  ```

- For a real report processor, you can make use of the `self.resource_statuses` and `self.metrics` hashes for details of resource status.
- Add this report processor to `[module]/lib/puppet/reports/`.
- Enable the report processor by adding its name to the list of processors in `puppet.conf`.

  ```ini
  [master]
    reports = store, puppetdb, logActionTime
  ```