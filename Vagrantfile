# -*- mode: ruby -*-
# vi: set ft=ruby :

PROJECT_NAME = ENV['PROJECT_NAME'] || File.basename(File.expand_path(File.dirname(__FILE__)))

###
# Variables that can be overriden on vagrant up.
# example:
#   ANSIBLE_TAGS=java vagrant provision --provision-with ansible
# Only run the ansible java tag in playbook.yml and the vagrant ansible provisioner.
###
VB_GUEST = ENV['VB_GUEST'] == 'true' ? true : false                                       # True runs the vagrant vb-guest plugin to update virtubalbox guest additions. Guest additions version needs to be older than the version of virtualbox.
ANSIBLE_VERSION = ENV['ANSIBLE_VERSION'] || '2.4.2.0'                                     # Version of ansible install on the guest
ANSIBLE_VERBOSITY = ENV['ANSIBLE_VERBOSITY'] || 'v'                                       # Set to vv to get debugging
ANSIBLE_TESTS_VERBOSITY = ENV['ANSIBLE_TESTS_VERBOSITY'] || 'vv'                          # Higher level of verbosity for tests.yml playbook.
ANSIBLE_GALAXY_FORCE = ENV['ANSIBLE_GALAXY_FORCE'] || "--force"                           # Set to "" to not force ansible galaxy run
ANSIBLE_TAGS = ENV['ANSIBLE_TAGS'] || "all"                                               # Set to specific ansible tags to run a subset of tasks and roles.
USERNAME = ENV['USERNAME']                                                                # Windows username passed as a variable into ansible.
VM_CPUS = ENV['VM_CPUS'] || 1                                                             # Number of CPUs for the guest VM.
VM_CPU_CAP = ENV['VM_CPU_CAP'] || 100                                                     # CPU usage cap on the guest OS.
VM_MEMORY = ENV['VM_MEMORY'] || 1024                                                      # RAM on the guest VM.
VM_GUI = ENV['VM_GUI'] == 'true' ? true : false                                           # Whether or not to show the VM GUI.
VM_DISKSIZE = ENV['VM_DISKSIZE'] || '20GB'                                                # Size of the root mount.
HOST_HTTP_PORTFORWARD_PORT = ENV['HOST_HTTP_PORTFORWARD_PORT'] || 8090                    # Port to forward to guest 8080
HOST_LOCALSTACK_PORTFORWARD_PORT = ENV['HOST_LOCALSTACK_PORTFORWARD_PORT'] || 4567        # Start of port range to forward to guest for localstack
SSH_KEY_NAME =  ENV['SSH_KEY_NAME'] || 'id_rsa'                                           # Name of your ssh key pair

###
# Install plugin dependencies if they don't exist.
###
plugins_dependencies = [['vagrant-reload', '0.0.1'], ['vagrant-vbguest', '0.16.0'], ['vagrant-cachier', '1.2.1'],
                        ['vagrant-disksize', '0.1.3']]
plugin_status = false
plugins_dependencies.each do |plugin|
  unless Vagrant.has_plugin? plugin[0]
    system("vagrant plugin install #{plugin[0]} --plugin-version #{plugin[1]}")
    plugin_status = true
    puts " #{plugin[0]} v#{plugin[1]}  Dependencies installed"
  end
end

## Restart Vagrant if any new plugin installed
if plugin_status === true
  exec "vagrant #{ARGV.join' '}"
else
  puts "All Plugin Dependencies already installed"
end

###
# The main section of the file. We do the following:
# - Configure the vagrant plugins that were installed above.
# - Do port forwarding to be able to access services running in the guest from the host.
# - Copy config files from the host to the guest.
# - Provision the guest VM using the ansible local provisioner.
# - Reload the VM so bash related changes get set for testing.
# - Run the ansible/test.yml playbook for basic verification.
###
Vagrant.configure("2") do |config|

  config.vm.box = "ubuntu/bionic64"

  ## Configure vagrant plugins
  if Vagrant.has_plugin?("vagrant-disksize")
    config.disksize.size = "20GB"
  end

  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.auto_update = VB_GUEST
  end

  if Vagrant.has_plugin?("vagrant-cachier")
    config.cache.scope = :box
  end

  ## Port forwarding
  # http for testing apps running locally on Guest VM.
  config.vm.network "forwarded_port", guest: 8080, host: HOST_HTTP_PORTFORWARD_PORT.to_i
  # Forwarded ports for localstack
  for i in HOST_LOCALSTACK_PORTFORWARD_PORT.to_i...HOST_LOCALSTACK_PORTFORWARD_PORT.to_i + 17
    config.vm.network :forwarded_port, guest: i, host: i
  end

  ## Set virtualbox provider configurations
  config.vm.provider :virtualbox do |vb|
    vb.gui = VM_GUI
    vb.cpus = VM_CPUS
    vb.customize ["modifyvm", :id, "--cpuexecutioncap", VM_CPU_CAP]
    vb.customize ["modifyvm", :id, "--memory", VM_MEMORY]
  end

  ## Copy config files from the host to the guest.
  config.vm.synced_folder '../', '/vagrant', disabled: false
  config.vm.synced_folder "~/.aws", "/home/vagrant/.aws", disabled: false
  config.vm.provision "file", source: "~/.gitconfig", destination: "/home/vagrant/.gitconfig"
  config.vm.provision "file", source: "~/.ssh/#{SSH_KEY_NAME}", destination: "/home/vagrant/.ssh/#{SSH_KEY_NAME}"
  config.vm.provision "shell",
                      inline: "chmod 0600 /home/vagrant/.ssh/#{SSH_KEY_NAME}"
  config.vm.provision "file", source: "~/.ssh/#{SSH_KEY_NAME}.pub", destination: "/home/vagrant/.ssh/#{SSH_KEY_NAME}.pub"
  config.vm.provision "shell",
                      inline: "chmod 0600 /home/vagrant/.ssh/#{SSH_KEY_NAME}.pub"

  ## Provision the guest VM using the ansible local provisioner.
  config.vm.provision "ansible", type: :ansible_local do |ansible|
    ansible.verbose = ANSIBLE_VERBOSITY
    ansible.install_mode = "pip"
    ansible.version = ANSIBLE_VERSION
    ansible.playbook = "#{PROJECT_NAME}/ansible/playbook.yml"
    ansible.galaxy_role_file = "#{PROJECT_NAME}/ansible/requirements.yml"
    ansible.galaxy_roles_path = "#{PROJECT_NAME}/ansible/roles"
    ansible.galaxy_command = "ansible-galaxy install --ignore-certs --role-file=%{role_file} --roles-path=%{roles_path} #{ANSIBLE_GALAXY_FORCE}"
    ansible.become = true
    ansible.extra_vars = {
        username: "#{USERNAME}".downcase
    }
    ansible.tags = ANSIBLE_TAGS
  end

  ## Reload the VM so bash related changes get set for testing.
  config.vm.provision :reload

  ## Run the ansible/test.yml playbook for basic verification. TODO This is sufficient and flexible enough. It could
  # be replaced by a test framework like GOSS or the tasks in playbook.yml could be refactored out to a proper ansible
  # role and the molecule testing framework used.
  config.vm.provision "ansible_tests", type: :ansible_local do |ansible|
    ansible.verbose = ANSIBLE_TESTS_VERBOSITY
    ansible.playbook = "#{PROJECT_NAME}/ansible/tests.yml"
    ansible.become = true
    ansible.tags = ANSIBLE_TAGS
  end

end