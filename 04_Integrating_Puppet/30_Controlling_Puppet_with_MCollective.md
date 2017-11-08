# Controlling Puppet with MCollective

- MCollective can be used to:
  - Query, stop, start, and restart the Puppet agent.
  - Run the Puppet agent with special command-line options.
  - Query and make changes to the node using Puppet resources.
  - Choose nodes to act on based on Puppet classes or facts on the node.
  - Control concurrency of Puppet runs across a group of nodes.

## Configuring MCollective

### Enabling the Puppet Labs Repository

- Install the repository as follows:

  ```bash
  sudo yum install http://yum.puppetlabs.com/puppetlabs-release-el-7.noarch.rpm
  ```

### Installing the MCollective Module

- Install the MCollective module from Puppet Forge.

  ```bash
  cd /etc/puppetlabs/code/environments/production
  puppet module install jorhett-mcollective --modulepath=modules/
  ```

### Generating Passwords

- Four strings are required for authentication.
  1. __Client Password:__ Used by the MCollective client to authenticate with the middleware.
  1. __Server Password:__ Used by nodes to authenticate with the middleware.
  1. __Preshared Salt:__ Used by clients and servers to validate that requests have arrived intact.
  1. __Java Keystore Password:__ Use to protect a Java keystore.
- The server credentials installed on nodes will allow them to subscribe to command channels, but not send commands.
  - If you use the same credentails for clients and servers, anyone with access to a server's configuration file will have control of the entire collective.
- Create 4 long, random strings, and copy them to a text editor.

  ```bash
  openssl rand -base64 32
  openssl rand -base64 32
  openssl rand -base64 32
  openssl rand -base64 32
  ```

- The pre-shared key model will work, but is only useful for demonstrative purposes.
  - TLS security plugins can be used to encrypt the transport of keys for full cryptographic authentication.

### Configuring Hiera for MCollective

- For testing, put the following Hiera configuration in `common.yaml`.

  ```yaml
  # Every node installs the server.
  classes:
    - mcollective::server

  # The Puppet Server will host the middleware.
  mcollective::hosts:
    - 'puppet.example.com'
  mcollective::collectives:
    - 'mcollective'
  mcollective::connector: 'activemq'
  mcollective::connector_ssl: true
  mcollective::connector_ssl_type: 'anonymous'

  # Access passwords.
  mcollective::server_password: 'Server Password'
  mcollective::psk_key: 'Pre-Shared Salt'

  mcollective::facts::cronjob::run_every: 10
  mcollective::server::package_ensure: 'latest'
  mcollective::plugins::agents:
    puppet:
      version: 'latest'

  mcollective::client::unix_group: vagrant
  mcollective::client::package_ensure: 'latest'
  mcollective::plugins::clients:
    puppet:
      version: 'latest'
  ```

- Every node will install and enable `mcollective::server`.
  - The remaining values identify the connection type.

### Enabling the Middleware

- For the purpose of testing, we can install the middleware on the puppetserver VM.
  - Requires minimal resources.
- The firewall on the middleware node must allow connections on 61614.

  ```bash
  sudo firewall-cmd --permanent --zone=public --add-port=61614/tcp
  sudo firewall-cmd reload
  ```

- Add Hiera data for the Puppet server that sets the MCollective classes and configuration.

  **/etc/puppetlabs/code/hieradata/hostname/puppetserver.yaml**

  ```yaml
  classes:
    - mcollective::middleware
    - mcollective::client

  mcollective::client_password: 'Client Password'
  mcollective::middleware::keystore_password: 'Keystore Password'
  mcollective::middleware::truststore_password: 'Keystore Password'
  ```

- Installs ActiveMQ middleware and client on the Puppet server.
- Run Puppet to configure the node.

  ```bash
  sudo puppet agent --test
  ```

### Connecting MCollective Servers

- Each node must run the Puppet agent again to configure MCollective.
- Verify that MCollective has established a connection with the Puppet server.

  ```bash
  netstat -an | grep 61614
  ```

### Validating the Installation

- All nodes should be online and connected to the middleware.
  - This can be verified with the `mco ping` command on the Puppet server.
- Troubleshooting
  - Verify the middleware host allows connections on port 61614.
  - The middleware host doesn't hav the same server password as the nodes.
    - Check `/var/log/messages` on the servers for authentication errors.
  - The middleware host doesn't have the same client password as your client.
    - Ensure the same value for `client_password` is used on both the client and middleware host.

### Creating Another Client

- You only need to install the client software on systems from which you will be sending requests.
  - Management hosts, or an administrator workstation.
- Create a per-host Hiera data file, and set the MCollective class/configuration.

  **/etc/puppetlabs/code/hieradata/hostname/client.yaml**

  ```yaml
  classes:
    - mcollective:client

  mcollective::client_password: 'Client Password'
  ```

- Run Puppet agent to enable MCollective.
- You can optionally tune which group has read access to the `client.cfg` file, reducing risk of malicious users obtaining the salt used to validate requests to the hosts.

  ```yaml
  mcollective::client::unix_group: 'vagrant'
  ```

### Installing MCollective Agents and Clients

- The MCollective Puppet agent is installed automatically using the previously mentioned Hiera data files.
  - Ensure the Puppet agent plugin is installed on all servers, and the Puppet client plugin is installed on all clients.

  ```yaml
  mcollective::plugins::agents:
    puppet:
      version: 'latest'

  mcollective::client::unix_group: vagrant
  mcollective::client::package_ensure: 'latest'
  mcollective::plugins::clients:
    puppet:
      version: 'latest'
  ```

### Sharing Facts with Puppet

- Node facts can be made available to MCollective.
  - Useful to filter the list of nodes that should act on a request.
- Set the `mcollective::facts::cronjob::run_every` parameter.
  - Number of minutes between updates of the file.
  - This causes `/etc/puppetlabs/mcollective/facts.yaml` to be populated with all Facter and Puppet-provided facts.
- You can use the `mco inventory` request to read through all facts available on a node.

  ```bash
  mco inventory client.example.com | awk '/Facts:/','/^$'
  ```

- You can query for how many nodes share the same value for facts.

  ```bash
  $ mco facts operatingsystem
  Report for fact: kernel

      Linux    found 2 times
      FreeBSD  found 1 times

  Finished processing 3 / 3 hosts in 61.45 ms
  ```

## Pulling the Puppet Strings

### Viewing Node Inventory

- This command allows you to see how a given server is configured.
  - What collectives it is part of.
  - What facts it has.
  - What Puppet classes are applied.
  - Running statistics.

  ```bash
  mco inventory client.example.com
  ```

- You can pass ruby scripts to the inventory service as well, for formatting the output in different ways.

  **~/inventory.mc**

  ```ruby
  # Format: hostname: architecture, operating system, OS release.
  inventory do
    format "%20s: %8s %10s %20s"
    fields {[
      identity,
      facts["os.architecture"],
      facts["operatingsystem"],
      facts["operatingsystemrelease"]
    ]}
  end
  ```

- Call the inventory command and pass the script file.

  ```bash
  mco inventory --script inventory.mc
  ```

### Checking Puppet Status

- You can query the Puppet agent status of any node using MCollective.
- First, find a list of nodes with the MCollective Puppet agent installed.

  ```bash
  mco find --with-agent puppet
  ```

- Now check the currnt agent status.

  ```bash
  mco puppet count
  ```

- A graphical summary of nodes can be output as well.

  ```bash
  mco puppet summary
  ```

### Disabling the Puppet Agent

- You may wish to disable the Puppet agent during maintenance periods, such as patching.
- An optional message can be added to explain why the disable took place.

  ```bash
  mco puppet disable --with-identity client.example.com message="Patching"
  ```

- If someone tries to run Puppet on the node, they will get a message back explaining that it is disabled.

  ```bash
  mco puppet runonce --with-identity client.example.com
  ```

- Once ready, the agent can then be re-enabled.

  ```bash
  mco puppet enable --with-identity client.example.com
  ```

### Invoking Ad Hoc Puppet Runs

- The Puppet agent can be instructed to evaluate the catalog immediately on a node.

  ```bash
  mco puppet runonce --with-identity client.example.com
  ```

- You can also target multiple nodes by fact values.

  ```bash
  mco puppet runonce --tags=sudo --with-fact operatingsystem=CentOS
  ```

- Or, you can run all servers in batches specified by a number input.

  ```bash
  # Run all, 5 at a time.
  mco puppet runall 5
  ```

- If there is an issue with overlap during the above, a sleep time can be introduced between batches when calling the `--batch` option instead of `runall`.

  ```bash
  mco puppet --batch 10 --batch-sleep 60 --tags ntp
  ```

### Limiting Targets with Filters

- Filters are used by the discovery plugin to limit which nodes are sent a request.
- Filters can be applied to any MCollective command.
- __Example:__ List all nodes with "serv" in the host FQDN.

  ```bash
  mco find --with-identity /serv/
  ```

- __Example:__ List hosts with the Puppet class mcollective::client.

  ```bash
  mco find --with-class mcollective::client
  ```

- __Example:__ List hosts with the operatingsystem fact set as Ubuntu.

  ```bash
  mco find --with-fact operatingsystem=Ubuntu
  ```

- There are two types of combination filters.
  - Combine Puppet classes and Facter facts.
    - __Example:__ Find all CentOS hosts with the puppet class 'nameserver' applied.

      ```bash
      mco find --with "/nameserver/ operatingsystem=CentOS"
      ```
  - Create searches with the select filter against facts and Puppet classes.
    - Includes Boolean logic.
    - __Example:__ All CentOS hosts that are not in the test environment.

      ```bash
      mco find --select "operatingsystem=CentOS and not environment=test"
      ```

    - __Example:__ Virtualized hosts with either the httpd or nginx Puppet class applied.

      ```bash
      mco find --select "(/httpd/ or /ngnx/) and is_virtual=true"
      ```

### Providing a List of Targets

- You can specify which nodes to make requests of using a file with one FQDN per line.

  ```bash
  mco puppet runonce --disc-method flatfile --disc-option /tmp/list-of-names.host
  ```

- Alternatively, you can pipe the list of nodes to the command:

  ```bash
  cat list-of-hosts.txt | mco puppet runonce -disc-method stdin
  ```

### Limiting Concurrency

- By default, every node which matches the query will respond immediately on contact.
- Flags in the command can be used to set how many servers receive a request in a batch, and how much time to wait between batches.

  ```bash
  # Requests a response from one (effectively rancom) node.
  mco puppet status --one
  ```

  ```bash
  # A fixed number of servers or a percentage of servers matching a filter.
  mco puppet status --limit 2
  ```

  ```bash
   # One-third of nodes with the webserver Puppet class applied.
   # Tell them to return their FQDN.
   mco shell run "hostname -fqdn" --limit 33% --with-class webserver.
   ```

### Manipulating Puppet Resource Types

- You can use MCollective to interact directly with each node's RAL.

  ```bash
  mco puppet resource service httpd ensure=stopped --with-identity /dashserver/
  ```

- The normal rules for using filters apply.