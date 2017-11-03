# Tracking Puppet Status with Dashboards

## Using Puppet Dashboard

- Web interface to browse the results of Puppet runs on nodes.
- Stores node reports in a database.
- Can also be used as an ENC.
  - Group nodes and associate them with classes and parameters.

### Intalling Dashboard Dependencies

- Name a node that hosts the dashboard with the same name as the serivce.
- Puppet Dashboard gets its own certificate, signed by the Puppet server.
  - This is separate from the key and certificate issued to the agent.

#### Starting the Dasboard VM

- Start up the `dashboard` Vagrant machine.

  ```bash
  vagrant up dashboard
  vagrant ssh dashboard
  sudo yum install -y git rsync vim
  ```

#### Preparing the Database

- Install the MariaDB fork of MySQL.

  ```bash
  sudo yum install -y mariadb-server
  ```

- Enable security for the database and run the installation.

  ```bash
  $ sudo systemctl start mariadb
  $ /usr/bin/mysql_secure_installation
  # Follow prompts to install.
  Set root password? [Y/n] y
  Remove anonymous users? [Y/n] y
  Disallow root login remotely? [Y/n] y
  Remove test database and access to it? [Y/n] y
  Reload privilege tables now? [Y/n] y
  ```

- Next, create a dashboard user, database, and table.

  ```bash
  mysql -u root -p
  # Enter the password set in the last step.
  MariaDB [(none)]> CREATE DATABASE dashboard_production CHARACTER SET utf8;
  MariaDB [(none)]> GRANT all ON dashboard_production.* TO dashboard@localhost IDENTIFIED BY '[password]';
  MariaDB [(none)]> quit
  sudo sed -i.bak /etc/my.cnf.d/server.cnf -e 's/\[mysqld\]/[mysqld]\nmax_allowed_packet = 32M/'
  sudo systemctl restart mariadb
  sudo systemctl status mariadb
  ```

#### Ensuring Dependencies

- The reason it is recommended to put Puppet Dashboard on a separate server is because of a large number of dependencies.
- First, the EPEL repository must be installed.

  ```bash
  sudo yum intall -y epel-release
  ```

- Ruby, RubyGems, Rake, and Bundler must be installed.

  ```bash
  sudo yum -y install ruby ruby-devel rake rubygem-bundler
  ```

- Compilers and a JavaScript runtime must be installed to build native binaries for some extensions.

  ```bash
  sudo yum install -y gcc-c++ nodejs
  ```

- The development libraries for your chosen database platform must also be installed.

  ```bash
  sudo yum install -y mariadb-devel libxml2-devel libxslt-devel sqlite-devel
  ```

#### Installing Apache for Puppet Dashboard

- To run the dashboard under Passenger, Apache will be used to provide the base web service.

  ```bash
  sudo yum install -y httpd httpd-devel mod_ssl
  ```

#### Installing Passenger for Puppet Dashboard

- Phusion provides a Yum repository with Passenger binaries.

  ```bash
  curl -sSLo passenger.repo https://oss-binaries.phusionpassenger.com/yum/definitions/el-passenger.repo
  sudo mv passenger.repo /etc/yum.repos.d/passenger.repo
  sudo yum install -y passenger mod_passenger
  ```

- Start Apache to confirm the Passenger configuration is correct.

  ```bash
  sudo systemctl enable httpd
  sudo systemctl start httpd
  sudo passenger-config validate-install --validate-apache2 --auto
  ```

#### Configuring Firewall for Puppet Dashboard

- Puppet servers need to connect to the dashboard to deliver reports.
- Users will also need to be able to browse the dashboard web console.

  ```bash
  sudo firewall-cmd --permanent --zone=public --add-port=80/tcp
  sudo firewall-cmd --permanent --zone=public --add-port=443/tcp
  sudo firewall-cmd --permanent --zone=public --add-port=3000/tcp
  sudo firewall-cmd --reload
  ```

- In a production setting, you would want to secure these ports to specific IP networks.

#### Enabling the Puppet Agent on Puppet Dashboard

- Even the Puppet dashboard node should be configured and managed via Puppet.

   ```bash
   puppet agent --test --server=puppet.example.com
   # Sign the cert from the Puppet server.
   # Re-run `puppet agent`.
   ```

### Enabling Puppet Dashboard

- The VM is now ready for installation of the Puppet Dashboard application.

#### Installing Puppet Dashboard

- Create the `puppet-dashboard` user account.
  - Most commonly, the user's home directory is placed in `/usr/share/puppet-dashboard` or `/opt/puppet-dashboard`.

    ```bash
    sudo useradd -d /opt/puppet-dashboard -m puppet-dashboard
    ```

- The remainder of the installation steps should be performed as the `puppet-dashboard` user.

  ```bash
  sudo su - puppet-dashboard
  chmod 755 /opt/puppet-dashboard
  rm .[a-z]*
  ls -la
  # The directory must be empty for the next step.
  git clone https://github.com/sodabrew/puppet-dashboard.git ./
  ```

- Next, the Ruby gems that the dashboard application is dependent on must be installed.
  - Bundler should be used so that the gems are installed in the `puppet-dashboard` user's home directory, thus not affecting system paths.

    ```bash
    bundle install --deployment --without postgresql
    ```

#### Configuring Puppet Dashboard

- Two configuration files are required:

  **/opt/puppet-dashboard/config/database.yml**

  ```yaml
  production:
    adapter: mysql2
    database: dashboard_production
    username: dashboard
    password: [password]
    encoding: utf8
  ```

  ```bash
  # Ensure this file is only readable by the dashboard user.
  chmod 0440 database.yaml
  ```

  **/opt/puppet-dashboard/config/settings.yml**

  ```bash
  # Copy this from the example file.
  cp settings.yml.example settings.yml
  ```

  - In `settings.yml` you must change the below five settings.

    ```yaml
    cn_name: 'dashboard.example.com'
    ca_server: 'puppet.example.com'
    inventory_server: 'puppet.example.com'
    file_bucket_server: 'puppet.example.com'
    disable_legacy_report_upload_url: true
    ```

    - Additionally, remove the `secret_token` line, as you will be generating a new one.

#### Defining the Puppet Dashboard Schema

- There is a rake task that can be used to define the database schema suitable for Puppet Dashboard.

  ```bash
  bundle exec rake db:setup
  bundle exec rake assets:precompile
  ```

#### Connecting Puppet Dashboard to Puppet Server

- Generate a unique TLS key and certificate request.
  - The CSR will be sent to the Puppet CA for signing.

  ```bash
  bundle exec rake cert:create_key_pair
  bundle exec rake cert:request
  # Log into the Puppet server and sign the certificate request.
  bundle exec rake cert:retrieve
  ```

- At this point, work as the `puppet-dashboard` user is complete, so we can disable its shell access.

  ```bash
  exit
  sudo usermod -s /bin/nologin puppet-dashboard
  ```

#### Enabling the Dashboard Rails Service

- Remove the unnecessary configuration files and install the provided definition file.

  ```bash
  cd /etc/httpd/conf.d
  sudo rm ssl.conf welcome.conf userdir.conf
  sudo cp /vagrant/etc-puppet/dashboard.conf ./
  sudo systemctl restart httpd
  sudo systemctl status httpd
  ```

#### Viewing the Dashboard

- If you are running the dashboard from Vagrant, you will need to add the node's IP address to your hosts file on your local machine.

  ```ini
  192.168.250.7 dashboard.example.com
  ```

- You can ignore the TLS certificate error for the purpose of this testing.

#### Sending Node Reports to the Dashboard

- Enable the dashboard as a target for node reports on your Puppet server in `/etc/puppetlabs/puppet/puppet.conf`.

  ```ini
  [master]
    reports = store, http
    reporturl = https://dashboard.example.com:3000/reports/upload
  ```

- After the Puppet server is restarted, it will forward node reports to the dashboard.

  ```bash
  sudo systemctl restart puppetserver
  ```

- Once Puppet server is back up, start a Puppet run on a node.
  - After this, a new report will be processed in the dashboard.

#### Enabling Worker Processes

- Puppet Dashboard uses worker processes to handle node reports.
  - A service must be enabled to run workers.
  - From the dashboard node, adjust the configuration files for this service.

  ```bash
  cd /opt/puppet-dashboard/ext/redhat
  sed -e 's#usr/share#opt#' puppet-dashboard.sysconfig | sudo tee /etc/sysconfig/puppet-dashboard
  echo "WORKERS=$(grep cores /proc/cpuinfo | cut -d' ' -f3)" | sudo tee /etc/sysconfig/puppet-dashboard-workers
  ```

- This adjust the system configuration files for the dashboard installation path, and sets the number of workers based on the number of cores.

  ```bash
  sudo systemctl enable puppet-dashboard-workers
  sydo systemctl start puppet-dashboard-workers
  ```

- Now that workers are enabled, pending reports should drop to 0.

### Viewing Node Status

- Now, any time a node converges, you will be able to view status on the Puppet Dashboard application.

### Using Dashboard as a Node Classifier

- The dashboard provides a GUI for setting classes and input parameters for nodes.
  - No history is retained!
- To enable this functionality, ensure that `enable_read_only_mode` is set to `false` in `/opt/puppet-dashboard/config/settings.yml`.
  - After changing this, make sure to restart Apache services.
- Generally, you will want to assign classes/parameters to node groups, not individual nodes.
  - Node groups can include nodes, other groups, classes that apply to all nodes in a group, and parameters provided with the group.

### Implementing Dashboard in Production

- The Puppet Dashboard service listening on port 3000 has rules to allow only connections that use TLS certificates signed by the Puppet CA.
- For user access on port 443, you will want to obtain a TLS certificate from a trusted CA, as with any public website.
  - Update the `SSLCertificateFile` and `SSLCertificateKeyFile` properties in `/etc/httpd/conf.d/dashboard.conf`.
- There is a built in job that optimizes the dashboard's database schema, which should be enabled via cron to run once a month.

  ```bash
  sudo crontab -u puppet-dashboard -e
  # Add the below.
  # * 3 1 * * /bin/bundle exec rake db:raw:optimize
  ```
- Old reports should be removed from the database regularly to prevent it from consuming all available space.
  - Add the following to the `puppet-dashboard` user's crontab.

  ```cron
  * 4 * 0 * /bin/bundle exec rake reports:prune upto=3 unit=mon
  ```

- Install the dashboard's logrotate file, which tells the `logrotate` process how to trim files in `./log/` of the installation directory.

  ```bash
  cd /opt/puppet-dashboard/ext/redhat
  sed -e 's#use/share#opt#' puppet-dashboard.logrotate | sudo tee /etc/logrotate.d/puppet-dashboard
  ```

- The dashboard is very write-heavy, so fast storage will significantly impact performance.
- Tune disk storage according to the below calculations.
  - This is not exact!

    | Nodes | Report Size | Puppet Runs | Purge Time | DB Size Adj | Expected Disk Space |
    |-------|-------------|-------------|------------|-------------|---------------------|
    | 50    | 128 KB      | 48          | 30         | 1.2         | 12 GB               |
    | 100   | 128 KB      | 48          | 90         | 1.2         | 67 GB               |
    | 2,000 | 256 KB      | 48          | 90         | 1.2         | 2.5 TB              |

## Upgrading to the Enterprise Console

- Puppet Enterprise provides a full-featured dashboard within the enterprise console.
- The PE console uses PuppetDB instead of a separate database.
- Provides all original functionality, and some additional features.

### Classifying Nodes

- In addition to grouping nodes and assigning classes/parameters, rules can use facts and metadata provided by the node to dynamically assign nodes to groups, and override the environment used by the node.

### Controlling Access

- RBAC based.
- Users and groups can be defined in the console, or imported from an external authentication mechanism, such as LDAP or MS AD.