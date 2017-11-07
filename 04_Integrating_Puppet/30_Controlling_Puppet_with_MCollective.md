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

### Checking Puppet Status

### Disabling the Puppet Agent

### Invoking Ad Hoc Puppet Runs

### Limiting Targets with Filters

### Providing a List of Targets

### Limiting Concurrency

### Manipulating Puppet Resource Types

## Comparing to Puppet Application Orchestration