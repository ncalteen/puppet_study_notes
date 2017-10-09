# Creating a Learning Environment

- Install VirtualBox and Vagrant
- If you are using OSX, install XCode as well.
- From the terminal, download the learning image.

  ```bash
  vagrant box add --provider virtualbox puppetlabs/centos-7.2-64.nocm
  ```

- Clone the learning repository.

  ```bash
  git clone https://github.com/jorhett/learning-puppet4
  ```

- Install the Vagrant vbguest plugin.
  - Ensures VirtualBox extensions on the VM are kept up to date.

  ```bash
  vagrant plugin install vagrant-vbguest
  ```

- Initialize the client system.

  ```bash
  cd ./learning-puppet4
  vagrant up client
  ```

- Verify Vagrant executes properly, as well as the status of your VMs.

  ```bash
  vagrant status
  vagrant suspend client
  vagrant resume client
  vagrant destroy client
  vagrant up client
  vagrant ssh client
  ```

- Verify the `/vagrant` filesystem.

  ```bash
  ls /vagrant
  ```

- Install utilities.

  ```bash
  sudo yum install -y rsync git
  ```