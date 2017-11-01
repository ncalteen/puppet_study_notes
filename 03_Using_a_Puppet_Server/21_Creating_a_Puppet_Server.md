# Creating a Puppet Server

## Starting the puppetserver VM

- Start the puppetserver Vagrant VM.
- Install needed packages.

  ```bash
  vagrant up puppetserver
  vagrant ssh puppetserver
  sudo yum install -y rsync git vim
  ```

## Installing Puppet Server

- Install the latest Puppet Labs Puppet Collection repository.
- Then, install the Puppet Server package.

  ```bash
  sudo yum install -y \
  > http://yum.puppetlabs.com/puppetlabs-release-pc1-el-7.noarch.rpm 
  sudo yum install -y puppetserver
  ```

## Configuring a Firewall for Puppet Server

- Puppet agents connect to servers on TCP port 8140 by default.
- The firewall on the puppetserver VM will need to be adjusted to allow connections on this port.

  ```bash
  sudo systemctl start firewalld.service
  sudo firewall-cmd --permanent --zone=public --add-port=8140/tcp
  sudo firewall-cmd --reload
  ```

- In a realistic setting, you would want to limit access to specific IP networks for this port.

## Configuring Puppet Server

- Puppet Server takes configuration from two places:
  - The primary configuration files at `/etc/puppetlabs/puppetserver/conf.d/`.
  - Historic values it reads from the `[master]` and `[main]` blocks of `/etc/puppetlabs/puppet/puppet.conf`.
  - For any setting available in the configuration files, the setting in `puppet.conf` will be ignored.
- __auth.conf:__ Authentication and authorization controls for Puppet Server and the Puppet CA.
- __global.conf:__ Global configuration settings common to all components of Puppet Server.
  - Contains only the location of the logback configuration file at this time.
- __puppetserver.conf:__ Configuration options for Puppet Server.
  - Some of these options overlap with options in `puppet.conf`.
  - If these two sources do not match, Puppet Server and the `puppet` command will use different directories.
- __webserver.conf:__ Web server configuration.
  - Authorization, TCP ports, and so on.
  - Supersedes the Apache or nginx configuration files being used wiht the old Puppet master Rack application.
- __web-routes.conf:__ Service mount points for Puppet Server's web applications.
  - Do not modify this file, as Puppet agents depend on specific mount points being available.
  - Should be managed only by the Puppet Server package.

### Defining Server Paths

- There are five variables which control what paths Puppet Server should use for files.
- These exist in two locations and must be kept in sync.

  | puppetserver.conf | puppet.conf | Default Value                              |
  |-------------------|-------------|--------------------------------------------|
  | master-conf-dir   | confdir     | `/etc/puppetlabs/puppet`                   |
  | master-code-dir   | codedir     | `/etc/puppetlabs/code`                     |
  | master-var-dir    | vardir      | `/opt/puppetlabs/server/data/puppetserver` |
  | master-run-dir    | rundir      | `/var/run/puppetlabs/puppetserver`         |
  | master-log-dir    | logdir      | `/var/log/puppetlabs/puppetserver`         |

- If these two files differ, then Puppet Server will use one location, while commands like `puppet certificate` will use another.

#### Enabling Use of the /var Filesystem

- The following change places all volatile and dynamic data within the `/var/` filesystem.

  **/etc/puppetlabs/puppet/puppet.conf**

  ```ini
  [user]
    vardir = /var/opt/puppetlabs/puppetserver
    ssldir = $vardir/ssl
  [master]
    vardir = /var/opt/puppetlabs/puppetserver
    ssldir = $vardir/ssl
  ```

  **/etc/puppetlabs/puppetserver/conf.d/puppetserver.conf**

  ```ini
  master-var-dir = /var/opt/puppetlabs/puppetserver
  ```

- Make sure to create the directory before starting puppetserver.

  ```bash
  mkdir -p /var/opt/puppetlabs/puppetserver
  chown -R puppet:puppet /var/opt/puppetlabs/puppetserver
  ```

### Limiting Memory Usage

- By default, Puppet Server tries to allocation 2GB of RAM.
- For testing, tune down the memory reserved by changing startup parameters.

  **/etc/sysconfig/puppetserver**

  ```ini
  JAVA_ARGS="-Xms512m -Xmx512m"
  ```

- This is a common issue with larger production environments, as memory may need to be raised.

### Configuring TLS Certificates

- Puppet Server accepts configuration of TLS kys and certificates in two locations.
- The following override settings can be defined in `/etc/puppetlabs/puppetserver/conf.d/webserver.conf`.
  - __ssl-cert:__ `/var/opt/puppetlabs/puppetserver/ssl/certs/puppet.example.com.pem`
  - __ssl-key:__ `/var/opt/puppetlabs/puppetserver/ssl/private_keys/puppet.example.com.pem`
  - __ssl-ca-cert:__ `/var/opt/puppetlabs/puppetserver/ssl/certs/ca.pem`
  - __ssl-crl-path:__ `/var/opt/puppetlabs/puppetserver/ssl/crl.pem`
- If none of these are defined, Puppet Server will fall back to using the settings from `/etc/puppetlabs/puppet/puppet.conf`.

  | puppetserver/conf.d/webserver.conf | puppet/puppet.conf |
  |------------------------------------|--------------------|
  | ssl-ca                             | localcacert        |
  | ssl-cert                           | hostcert           |
  | ssl-key                            | hostprivkey        |
  | ssl-crl-path                       | hostcrl            |

- If you use the settings in `webserver.conf`, you must use all four together.
- At this time, the Certificate Authority continues to use the following settings from `puppet.conf`.
  - __cacert:__ `/etc/puppetlabs/puppet/ssl/certs/ca.pem`
  - __cacrl:__ `/etc/puppetlabs/puppet/ssl/crl.pem`
  - __hostcert:__ `/etc/puppetlabs/puppet/ssl/certs/puppet.example.com.pem`
  - __hostprivkey:__ `/etc/puppetlabs/puppet/ssl/private_keys/puppet.example.com.pem`
- Its recommended to use `/etc/puppetlabs/puppet/puppet.conf` exclusively, and leave `webserver.conf` blank (this is the default when installed).
- You can see configuration values for the master and CA via the `sudo puppet config print --section master` command.

### Avoiding Obsolete Settings

- The following are valid for Puppet master, but have been replaced by other options in Puppet Server:
  - `autoflush`
  - `bindaddress`
  - `ca`
  - `keylength`
  - `logdir`
  - `masterhttplog`
  - `masterlog`
  - `masterport`
  - `ssl_client_header`
  - `ssl_client_verify_header`
  - `ssl_server_ca_auth`
- The following are valid for the Puppet master, but have no corresponding equivalent in Puppet Server.
  - `capass`
  - `caprivatedir`
  - `daemonize`
  - `http_debug`
  - `puppetdlog`
  - `railslog`
  - `syslogfacility`
  - `user`
- The following are valid for Puppet agents, but will be ignored by Puppet Server:
  - `configtimeout`
  - `http_proxy_host`
  - `http_proxy_port`

### Configuring Server Logs

- Uses the JDK logback library.
- Configuration file can be found at `/etc/puppetlabs/puppetserver/logback.xml`.
- By default, Puppet Server logs all messages at INFO or higher to `/var/log/puppetlabs/puppetserver/puppetserver.log`.
- By default, logback scans for configuration changes every minute.
  - Restarting Puppet Server is not needed.

### Configuring Server Authentication

- Puppet Server authentication and authorization is defined in `/etc/puppetlabs/puppetserver/conf.d/auth.conf`.

#### Understanding Authorization Rules

- Within each authorization block is a list of rules.
- Each rule contains a name, sort-order, match-request, and an allow/deny.
  - These are used to match with a node's certificate name.
- Multiple values for allow or deny should be written as an array.
- The asterisk matches any validated node.
- The allow-unauthenticated is used to allow new nodes to submit certificate requests and retrieve the signed certificate.

  ```hocon
  {
      # Allow nodes to retrieve only thier own node definition.
      match-request: {
          path: "^puppet/v3/node/([^/]+)$"
          type: regex
          method: get
      }
      # Only the same node can request its catalog
      allow: "$1"
      sort-order: 500
      name: "puppetlabs node"
  }
  ```

## Running Puppet Server

  ```bash
  sudo systemctl enable puppetserver
  sudo systemctl start puppetserver
  ```

- Check the startup log output:

  ```bash
  sudo tail -f /var/log/puppetlabs/puppetserver/puppetserver.log
  ```

### Adding Ruby Gems

- Puppet Server has its own Ruby installation, aside from any installed on the server or for the Puppet agent.

  ```bash
  # Identical to Ruby's gem command.
  sudo puppetserver gem list
  ```

- If you need other gems for a plugin running on the server, use `puppetserver gem` to install them.

  ```bash
  sudo puppetserver gem install bundler --no-ri --no-rdoc
  ```

- Use native Ruby gems wherever possible, as JRuby does not support gems that use binary or compiled extensions.

## IPv6 Dual-Stack Puppet Server

- Puppet Server uses IPv6 by default, without any additional settings.
- If you are upgrading a Puppet master, remove or comment out the `bindaddress` configuration setting to enabled IPv6 on a Puppet master.

  ```ini
  [master]
  #  bindaddress = ::
  ```

- If you check for listening services with `netstat -an | grep 8140` you will se that Puppet Server is listening on IPv4 and IPv6.