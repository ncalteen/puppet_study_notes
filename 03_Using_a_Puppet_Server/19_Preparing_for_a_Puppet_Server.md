# Preparing for a Puppet Server

- How does a Puppet server change the catalog build process?
- How do you build a Puppet server so it is easily moved or upgraded?
- Is Puppet Master or Puppet Server more appropriate for your nodes?

## Understanding the Catalog Builder

### Node

- A discrete system that could be managed by a Puppet agent.
  - A physical computer.
  - A virtualized OS.
  - A cloud-hosted virtualized OS.
  - Routers, switches, and VPN concentrators.
  - Hypervisors.
  - Containers.
  - Different users or paths on the same system.
  - The list goes on...

### Agent

- Evaluates and applies Pupept resources from the catalog on a node.
- The `puppet apply` process:
  1. Gather Node Data
     - Reads the `environment` value from the config file or command-line input.
     - Retrieves plugins including functions and facts.
     - Runs Facter to generate node facts (built-in and custom).
     - Selects the node name from the config file or fact data.
  1. Build a Catalog of Resources for the Node
     - Query a node terminus (if configured) to obtain node information.
     - Evaluate the main manifest, node terminus, and Hiera data to create a list of classes to apply.
     - Performs all variable assignment.
     - Evaluates `if`, `unless`, and other conditional attributtes.
     - Performs iterations to build resources or map values.
     - Executes functions configured in classes to provide module-specific data.
     - Executes functions configured in the environment to provide environment-specific data.
  1. Evaluates (Converges) the Catalog on the Node
     - Evaluates the `onlyif`, `creates`, and other conditional attributes to determine which resources in the catalog should be applied to the node.
     - Creates a dependency graph based on ordering attribute and automatic dependencies.
     - Compares the state of each resource, and makes the necessary chagnes to bring the resource into compliance with the policy.
     - Creates a report containing all events processed during the agent run.
- The Puppet agent itself needs only to process the catalog, and thus does not need to process compilation of said catalog.

### Server

- The sever takes over the process of evaluating data and compiling catalogs for nodes to evaluate.
  1. The Puppet agent submits name, environment, and facts to Puppet server.
  1. The Puppet server builds a Puppet catalog for the node.
  1. The Pupept agent applies the catalog's resources on the node.
- Only the server requires access to the raw Puppet modules and thier data sources.
  - Removes the need to distribute code and data to every node.
  - Reduces computational cost on the end node.
  - Provides auditing and controls for some security requirements.
  - Simplifies network access requirements for internal data sources.

## Planning for Puppet Server

### The Server is Not the Node

- The name of the Puppet server will exist for the lifetime of this environment.
- Do not confuse a node with the Puppet Server service.
- When you start the Puppet server services, it will create a new certificate authority with the name you assign.
  - Every Puppet agent that will use the service must acquire a certificate signed by this authority.
  - Renaming the service would require recertification of every node in the environment.
- The node on which Puppet server services run can be changed relatively easily.
  - Use a globally unique name for your Puppet server that can easily be moved or shared between multiple servers.
  - Set the following in `/etc/puppetlabs/puppet/puppet.conf`.

    ```ini
    [agent]
      server = puppet.example.com

    [master]
      certname = puppet.example.com
    ```

### The Node is Not the Server

- Do not set the `certname` of the node to the name of the Puppet server.
  - It should be distinct from the server.
- Every node that connects to the Puppet server should have a unique name and TLS key/certificate.
- On the node that hosts the Puppet server, configure Puppet agent to use the node's fully qualified and unique hostname.

### Store Server Data Files Separately

- The Puppet Server runs as a nonprivileged user account.
- The Puppet agent runs as a privileged user on the node.
- Puppet stores certificate data for its nodes within `/etc/puppetlabs/`.
  - For a managed node, this makes sense (`/etc/` should store static config files).
  - For a Puppet server, autoscaled environments could add and remove thousands of test nodes per day, making `/etc/puppetlabs/` very volatile.
- Better recommendation is to place volatile files within `/var`.

  **/etc/puppetlabs/puppet/puppet.conf**

  ```ini
  [user]
    vardir = /var/opt/puppetlabs/server
    ssldir = $vardir/ssl
  [master]
    vardir = /var/opt/puppetlabs/server
    ssldir = $vardir/ssl
  ```

- Default locations are:

  ```ini
  [master]
    vardir = /opt/puppetlabs/puppet/cache
    ssldir = /etc/puppetlabs/puppet/ssl
  ```

### Functions Run on the Server

- Functions are executed by the process that builds the catalog.
- Log messages will be in the server logs, not the node.
- Data used by the catalog must be local to the server, or supplied by facts.
- The only method to provide local node data to the server is via facts.
  - Functions that read data from the node must be changed to custom facts that supply the same data.

## Choosing Puppet Master Versus Puppet Server

### Upgrading Easily with Puppet Master

- The Puppet master is a Rack application.
- Comes with a built-in application server, WEBrick, which can accept only two concurrent connections.
- Anything larger than a few dozen nodes would hit limitations of this server.
- Scalable alternative is to run Puppet master under an application server such as Phusion Passenger.
  - Puppet 4's AIO installer makes this easier.
- Puppet master only supports Puppet 4 clients.
  - Any earlier versions will need to be upgraded.

### Embracing the Future with Puppet Server

- Puppet Server is a self-standing product intended to expand on Puppet master.
  - Drop in replacement for Puppet master.
- Uses Java, rather than Ruby, as the underlying engine.
- Supports Ruby module plugins through the JRuby interpreter.
- Provides more visibility into the server process via metrics available to analytics tools.
- Backwards compatible with Puppet 3 clients.

### Why There's Really No Choice

- Puppet master has been deprecated, and will not exist in Puppet 5.

## Ensuring a High-Performance Server

- Two issues to consider for performance: available RAM and CPUs.
- On a server with sufficient RAM for Puppet server threads, the Puppet modules and Hiera data will be cached in file buffer memory.
  - The only disk I/O will be to store reports, and when module or data changes are written out.
  - Generally 10:1 writes versus reads.
- Memory utilization tends to be very stable.
  - Increase memory as needed until the file cache becomes stable.
  - Any free memory not used by processes will be used for the file cache.
- Vast majority of the server's processing time will be spent compiling catalogs for nodes.
- Optionally, Puppet serves can be scaled horizontally (i.e. with Compile Masters).