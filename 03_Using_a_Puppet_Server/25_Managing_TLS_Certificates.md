# Managing TLS Certificates

## Reviewing Node Authentication

- __Bidirectional Validation:__ Both the Puppet server and agent validate the other's certificate.
  - Prevents man in the middle attacks.
- Once both sides have validated each other, they use the certificates to negotiate an encrypted communication session.
- When Puppet provides the CA, the process for how this occurs is:
  1. A CA server is configured to sign Puppet agent CSRs.
  1. Each Puppet server has a certificate signed by the Puppet CA.
  1. Each agent submits a CSR to the CA before attempting to contact a server.
  1. The CSR is signed manually via the `puppet cert` command, or via automatic signing.
  1. The agent retrieves the signed certificate, and then contacts the Puppet server.
  1. The Puppet server verifies the agent's certificate was signed by the same CA.
  1. The agent verifies that the Puppet server's certificate was signed by the same CA.

## Autosigning Agent Certificates

- Manual signing may not be viable in a heavily-scaled environment.
- Different types of automatic signing can be enabled for a CA.

### Name-Based Autosigning

- Automatically sign certificates based on the name given by the requestor.
- Set the value of `autosign` in the `[master]` section of `puppet.conf` to the name of a configuration file containing the certificate names that should be signed automatically.

  ```ini
  [master]
    autosign = $confdir/autosign.conf # Default value.
  ```

- The file should contain the full name or glob(*)-style wildcards that match certificates to sign.

  ```bash
  # Enable autosigning for every node in the example.com domain.
  # Will not match subdomains like web.server.example.com.
  echo "*.example.com" > /etc/puppetlabs/puppet/autosign.conf
  # Match a subdomain.
  echo "web.server.example.com" > /etc/puppetlabs/puppet/autosign.conf
  # Trailing * will not work (i.e. web.server.*)
  ```

- This does not provide security.
  - Any user without root access can generate a CSR as long as they know the valid naming options.
  - Should only be used if the Puppet server is controlled by firewalls and other security processes.

### Policy-Based Autosigning

- Automatically sign certificates based on response provided by an external program.
- Set the value of `autosign` in the `[master]` section of `puppet.conf` to the name of an executable program.

  ```ini
  [master]
    autosign = /path/to/decision/maker
  ```

- The program can be in any compiled or scripting language, so long as it is exectuable by the `puppet` user.
- It will receive the `certname` requested as the only command-line argument, and the contents of the CSR in PEM formet on STDIN.
- The program needs to return an exit code of success (0) to accept the CSR, or any nonzero to reject.
- If the program makes the decision based only on the name of the requestor, it is no more secure than name-based autosigning.
  - It should parse the PEM data to validate the nature of the request.
- External tools such as databases, APIs, and others can be leveraged to further validate the request.
  - I.E. Verify the machine was recently provisioned in AWS with this certname.
- Security of policy-based signing relies on the security of the data sources queried.

#### Adding Custom Data to CSRs

- You must perform the following steps on the node:
  - Create a YAML format file with the custom data.
  - Specify the YAML file wth the `csr_attributes` Puppet configuration setting.
  - Generate a new certificate for the node.
- Default location for the attributes file is `$confdir/csr_attributes.yaml`.
  - Must contain two keys, the value of which should be a hash containing the attributes to be provided.

  ```yaml
  ---
  # Custom attributes will be discarded when the certificate is signed.
  custom_attributes:
    2.999.1.3: "Custom value 513 in the documentation OID"
    pp_uuid: "A unique instance identifier to be validated"
  # Extension requests will be added to the final certificate
  # and available to the Puppet server during catalog build.
  extension_requests:
    pp_cost_center: "Custom value to be used in catalog build."
  ```

- Populate this file with any mechanism desired when configuring new nodes, such as AWS instance user data.
- Only after this file is populated should you attempt to connect to the Puppet server.
- OIDs (extension names) are not parsable without creating a custom mapping.
  - There are a number of built in OIDs which can be used and referenced by name.

    |                |                  |                     |                |
    |----------------|------------------|---------------------|----------------|
    | pp_application | pp_cluster       | pp_cost_center      | pp_created_by  |
    | pp_department  | pp_employee      | pp_environment      | pp_image_name  |
    | pp_instance_id | pp_preshared_key | pp_product          | pp_provisioner |
    | pp_role        | pp_service       | pp_software_version | pp_uuid        |

#### Inspecting CSRs

- The custom data in the CSR can be found in the `Attributes` or `Requested Extensions` block of the CSR.
  - Your program will need to parse the PEM formatted CSR.

#### Using Extension Requests in Puppet

- The data can be refrenced by the `extensions` hash key of the trusted node facts.

  ```puppet
  notify { 'cost-center':
    message => "Cost center is ${trusted['extensions']['pp_cost_center']}"
  }
  ```

### Naive Autosigning

- Trust any agent that connects to the Puppet server.
- Every CSR is immediately signed.
- Don't do this.
  - Ever

  ```ini
  [master]
    autosign = true
  ```

## Using an External Certificate Authority

- By default the Puppet server will generate its own TLS key and self-signed certificate, creating a new certification tree with it as the root authority.
- In large enterprises, it is often necessary to have all keys issued from a centralized authority outside of Puppet.
- Puppet servers support three configurations:
  - A Puppet server is the sole CA, and signs all certificates.
  - A single external CA signs certificates for both Puppet servers and agents.
  - Two external CAs sign certificates: one for Puppet servers, and one for agents.

### Distributing Certificates Manually

- With external CAs, Puppet has no way to help distribute certificate files between machines.
  - Will require an external solution to distribute certificates.
- Once you have the certificates, install them on the configured locations on each node.
  - To verify the location, run the following:

  ```bash
  sudo puppet config --section agent print hostcert hostprivkey localcacert
  ```

- If the certificates are issued by an intermediate CA, you will need to set a configuration value for the location of the intermediate CA's certificate.

  ```ini
  [agent]
    ssl_client_ca_auth = /path/to/intermediate/ca/certificate.pem
  ```

### Installing Certificates on the Server

- You are required to create a CSR for the Puppet server and get it signed by the external CA.
- Install the certificates in the configured locations (use the command above to verify).
  - Copy the Puppet server's key to the `hostprivkey` location.
  - Copy the CA's certificate to the `localcert` location.
  - Copy the certificate signed by the external CA to the `hostcert` location.

### Disabling CA on a Puppet Server

- Comment out the following line in `/etc/puppetlabs/puppetserver/bootstrap.cfg`:

  ```conf
  #puppetlabs.services.ca.certificate-authority-service/
    #certificate-authority-service
  ```

- Uncomment the following line:

  ```conf
  puppetlabs.services.ca.certificate-authority-disabled-service/
    certificate-authority-disabled-service
  ```

- Ensure that `/etc/puppetlabs/puppetserver/conf.d/webserver.conf` contains the following:

  ```conf
  webserver: {
      ssl-key       : /var/opt/puppetlabs/puppetserver/ssl/private_keys/puppet.example.com.pem
      ssl-cert      : /var/opt/puppetlabs/puppetserver/ssl/certs/puppet.example.com.pem
      ssl-ca-cert   : /var/opt/puppetlabs/puppetserver/ssl/ca/ca_crt.pem
      ssl-cert-chain: /var/opt/puppetlabs/puppetserver/ssl/ca/ca_crt.pem
      ssl-crl-path  : /var/opt/puppetlabs/puppetserver/ssl/crl.pem
  }
  ```

### Using Different CAs for Servers and Agents

- One intermediate CA signs certificates issued to Puppet servers, while another intermediate CA signs certificates issued to Puppet agents.
- This process is the same as using one external CA, with the exception of the following:
  - __Puppet Servers:__ The Puppet servers have the root CA certificate and intermediate CA certificate that signs Puppet agent keys in the file referenced by `localcert`.
    - This allows them to validate any agent certificate, but will fail to validate the certificate of another Puppet server.
  - __Puppet Agents:__ Puppet agents will need the certificate of the intermediate CA that signs keys for the Puppet servers installed in the location specified by the `ssl_client_ca_auth` setting.
- Puppet agent certificates cannot successfully validate on other Puppet servers, nor will Puppet servers accept keys from other Puppet servers.
- Useful when different teams control different Puppet servers that share the same root CA.

### Distributing the CA Revocation List

- You can enable CRL checking on both Puppet servers and agents.
- Install the CRL in the configured locations.
  - Default is `/etc/puppetlabs/puppet/ssl/crl.pem`.

  ```bash
  sudo puppet config --section agent print hostcrl
  ```

- Puppet CAs provide a CRL to all agents, which is retireved when the agent acquires its certificate back from the CA.
- If a revocation list is not provided by the external CA, you must disable CRL checking on the agent.

  ```ini
  [agent]
    certificate_revocation = false
  ```

- As the agent will attempt to download the CRL from the Puppet server, which has its CA disabled, the request will fail.

## Learning More About TLS Authentication

- Task One: Set Up Name-Based Autosigning
  - Configure the server to autosign certificates for `*.example.com`.
  - Run `vagrant up web2` and install Puppet.
  - Start Puppet agent on the instance and connect to the server.
  - Observe the Puppet agent immediately receive back a signed certifiate.
  - Observe the module converge without having to run Puppet again.
  - Review the node report in `$vardir/reports`.
- Task Two: Implement Policy-Based Autosigning
  - Create a script to inspect CSR attributes and exit with a nonzero exit code if a certain value isn't one.
  - Specify this program as the value of the `autosign` configuration variable on the Puppet server.
  - Run `vagrant up web3` and install Puppet.
  - Attempt to connect to the Puppet server and observe the rejection.
  - Regenerate the CSR with the custom attribute.
  - Attempt to connect again to the Puppet server.