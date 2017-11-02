# Connecting a Node

## Creating a Key Pair

- The easiest way to create a key pair is to attempt to connect to a server.
- The Puppet agent will create private and public TLS keys, and submit a CSR to the server for authorization.
- Verify the default hosts file on the client VM associates `puppet.example.com` with the puppetserver VM.

  ```bash
  cat /etc/hosts
  ```

- Attempt a connection from the client VM to the Puppet Server.

  ```bash
  puppet agent --test --server=puppet.example.com
  ```

- The client has done the following:
  - Create a private and public TLS key for itself.
  - Connected to the server and retrieved a copy of its TLS certificate.
  - Created a CSR for itself and submitted to the server.
- The client cannot proceed until it is authorized.

## Authorizing the Node

- The client's CSR must be authorized from the Puppet server.

  ```bash
  puppet cert --list
  puppet cert --sign client.example.com
  ```

- The next time the client connects, it will be given the signed certificate and a Puppet catalog will be compiled for it.

## Downloading the First Catalog

- Retry the connection attempt from the client node.

  ```bash
  puppet agent --test --server=puppet.example.com
  ```

- The agent submitted the node name, environment, and facts to the Puppet server.
- The Puppet master compiled a catalog for the node.
- The agent evaluated the catalog.

## Installing Hiera Data and Modules

- If you are building a new environment, copy the Hiera data, envirnoments, and modules from Part 2 from the client instance.

  ```bash
  sudo passwd vagrant
  # set a password
  ```

- On the puppetserver instance, recursively copy over the entire code-dir path.
  - Before doing this, you will need to set `PasswordAuthentication yes` in `/etc/ssh/sshd_config`, then restart the `sshd` service.

  ```bash
  sudo rsync -aH vagrant@client.example.com:/etc/puppetlabs/code /etc/puppetlabs/
  # Enter authentication information
  ```

## Testing with a Client Node

- At this point, you can test the same manifest on your client.
- The difference is that the Puppet Server will evaluate the manifest and biild the catalog, instead of the local Puppet agent.

  ```bash
  sudo puppet agent --test --server=puppet.example.com --environment=test
  ```