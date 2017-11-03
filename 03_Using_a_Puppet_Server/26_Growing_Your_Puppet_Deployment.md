# Growing Your Puppet Deployment

- This section contains topics and considerations needed for large Puppet deployments.

## Using a Node Terminus

- A node data provider for the Puppet server, which can do the following:
  - Override `environment` set by the node.
  - Declare classes and parameters to apply to the node.
  - Define top-scope variables.
- The following are available as options for a node terminus.
 - External Node Classifier (ENC)
 - LDAP Terminus
- ENCs are rarely needed, as the global and environment data providers have more data options than ENCs can provide.
  - Use an ENC only when you cannot retrieve data via Puppet Lookup.

### Running an External Node Classifier

- Provides environment, class, and variable assignments for nodes.
- Executed by the `exec` node terminus of the Puppet server prior to selecting the node's environment.
  - Called by the node that builds the catalog (Puppet server in server/agent setup, or node in `puppet apply` implementation).
- Can be any type of program.
  - Defined in `/etc/puppetlabs/puppet/puppet.conf`.

    ```ini
    [master]
      node_terminus = exec
      external_nodes = /path/to/node/classifier
      # Needs to be placed in [user] when using puppet apply.
    ```

- The program receives the value of `certname` as a command-line argument.
- The program should output YAML with `parameters` and `classes` hashes, and an optional `environment` string.
  - __environment:__ The environment to be used when building the node's catalog.
    - Overrides the environment configuration of the agent.
  - __parameters:__ Node variables declared in the top scope (i.e. `::parameter`).
    - Optional if `classes` hash is defined.
  - __classes:__ A hash of classes to be declared, much like the Hiera class definition.
    - Optional if `parameters` hash is defined.
    - You can indent input parameters below each class name.
- Example output showing single-level and multi-level paramter and class names.

  ```bash
  $ /path/to/node/classifier client.example.com
  ---
  environment: test
  parameters:
    selinux: disabled   # ::selinux
    time:
      timezone: PDT     # ::time::timezone
      summertime: true  # ::time::summertime
  classes:
    ntp:
    dns:
      search: example.com
      timeout: 2
      options: rotate
    puppet::agent:
      status: running
      enabled: true
  ```

- If no parameters or classes should be assigned, then return an empty hah as the result for one or both parameters.

  ```bash
  $ /path/to/node/classifier client.example.com
  ---
  classes: {}
  parameters: {}
  ```

### Querying LDAP

- Requires rights to modify LDAP schemas on the server.
- Add the following to `/etc/puppetlabs/puppet/puppet.conf`.

  ```ini
  [master]
    node_terminus = ldap
    ldapserver = ldap.example.com
    ldapport = 389
    ldaptls = true
    ldapssl = false
    ldapuser = readuser
    ldappassword = readonly
    ldapbase = ou=Hosts,dc=example,dc=com
  ```

- Download and apply the Puppet LDAP schema from Puppet's GitHub repository.
- You will need to add LDAP entries for each Puppet node.

  ```ini
  dn: cn=client,ou=Hosts,dc=example,dc=com
  objectClass: device
  objectClass: puppetClient
  objectClass: top
  cn: client
  environment: test
  puppetClass: dns
  puppetClass: puppet::agent
  puppetVar: selinux=disabled
  ```

- LDAP does not allow multilevel variables or parameters for classes.

## Deploying Puppet Servers at Scale

- Given the same module code, Hiera data, and access permissions, two different Puppet servers will render the same catalog for the same node every time.
- However, administrators will need to decide how to synchronize files and whether or not to centralize signing of TLS certificates.

### Keeping Distinct Domains

- Implement a Puppet server for each group of nodes.
- Simple when different teams manage different servers and nodes.
- Each Puppet server acts as its own CA.
- Each Puppet server can host the same modules, a distinct set of modules, or some combination thereof.

### Sharing a Single Puppet CA

- A single Puppet CA for all Puppet servers in the organization.
- Configure agents to submit all CSRs to this CA.

  ```ini
  [agent]
    ca_server = puppet-ca.example.com
    server = puppetserver01.example.com
  ```

- Every server and agent will get certificates from this CA.
- Nodes point to Puppet servers for all normal Puppet actions.
- You will need to sync all approved agent certificates from the CA down to each Puppet server.
  - You can use a NFS to share the server's TLS directory (`/var/opt/puppetlabs/puppetserver/ssl`), or use something like `rsync` to keep directories in sync.
  - If using NFS, you may want to share `/var/opt/puppetlabs/puppetserver/reports` so that all client reports are centrally stored.

### Using a Load Balancer

- It may be necessary to put multiple Puppet servers behind a load balancer.
- It is not recommended to enable TLS offloading on the load balancer.
  - The performance benefits do not outweight the configuration requirements to ensure it works as expected.
- Each server must have the same modules, Hiera data, configuration, etc., or they must mount it from the same place (i.e. a NFS share).
  - You would want to share the below directories between servers:
    - `/etc/puppetlabs/code/`: Puppet modules and Hiera data.
    - `/var/opt/puppetlabs/puppetserver/bucket/`: File backups from agent nodes.
    - `/var/opt/puppetlabs/puppetserver/jruby-gems/`: Ruby gems used by the server.
    - `/var/opt/puppetlabs/puppetserver/reports/`: Node reports.
    - `/var/opt/puppetlabs/puppetserver/tls/`: TLS keys and certs.
    - `/var/opt/puppetlabs/puppetserver/yaml/`: Node facts.

### Managing Geographically Dispersed Servers

- There are three ways to synchronize data on globally disparate Puppet servers.
- __Manually:__ This may work for a short time, but will become painful quickly.
  - Only viable when different teams share a minimal amount of Puppet code and data.
- __Centralized Source:__ Each Puppet server checks out the module code and data from a source repository.
  - Can be enabled by schedule, post-commit handler, manual authorization, or automated triggers using tools like MCollective.
  - Provides relatively fast synchronization of changes to all servers.
- __Rolling Push:__ Progressively deploy changes through staging and canary environments.
  - Automation schedulers are used to roll changes from one environment to another.
  - Enables a cautious, slow-rolling deployment.
- Deployment strategies should be modified to meet organizational needs.

### Managing Geographically Dispersed Nodes

- Nodes can be distributed globally and managed in two ways:
- __Servers Close to Nodes:__ Puppet servers placed in each local network with nodes.
  - Requires data synchronization between Puppet servers.
  - Can cause latency/inconsistency between regions.
- __Centralized Servers:__ A larger cluster of Puppet servers in a single location.
  - Connectivity between agents and servers can be inconsistent, and latency can tie up connections to servers for long periods of time.

### Falling Back to Cached Catalogs

- If a node loses connectivity to its Puppet server, you can decide whether or not the agent should run the last cached catalog.

  ```ini
  [agent] # Default values
    usecacheonfailure = true
    use_cached_catalog = false
  ```

- __usecacheonfailure:__ Allows the Puppet agent to use the last good catalog if the Puppet server doesn't respond with a new one.
- __use_cached_catalog:__ Tells the Puppet agent to *always* use the last cached catalog, not checking for a new one from the Puppet server.
  - Useful only when you wish to manually deploy all changes.

### Making the Right Choice

- No choice is exclusive.
- Combine the options that work best for the individual use case.

## Best Practices for Puppet Servers

- The server is not the node.
  - Name the server with an alias that points to the node hosting it.
- The node is not the Puppet server.
  - Configure the node as a normal Puppet agent.
  - This makes it easier to migrate later.
- Store the Puppet server's volatile data files in `/var/`.
- Don't use naive or unvalidated autosigning!
- Puppet Lookup is more flexible than ENCs.
  - Don't use an ENC unless it provides some content you cannot retrieve with Puppet Lookup.
- Don't over-tune the log levels of every resource.
  - Node reports provide direct access to every detail of a converge report.