# Using a Puppet Server

- Puppet server authenticates and provides centralized infrastructure for Puppet agents, including:
  - Distributes policies to nodes without file sync.
  - Evaluates and builds catalogs for the nodes.
  - Provides a central source for configuration data.
  - Collects convergence reports from Puppet agents.
  - Stores backups of files managed by Puppet.
- __Puppet Master:__ Refers to the Puppet server provided by the deprecated `puppet master` command.
  - Only server available in previous Puppet versions.
- __Puppet Server:__ Refers to the Puppet Server product, whic is packaged separately.
  - High-performance, scalable replacement for the Puppet master.