# Local Development VM

This role spins up a Ubuntu 18.04 development environment virtual machine that is provisioned and configured with 
ansible. All the devops developer tools are installed at specific versions. Look at ansible/playbook.yml for the 
versions of all tools installed.

It uses vagrant, virtualbox and make/gradle. 


# Requirements

The following needs to be installed on Windows host

- Virtualbox 5.2+
- Vagrant 2.0.1+
- ~/.ssh/id_rsa
- ~/.ssh/id_rsa.pub
- ~/.aws/config
- ~/.aws/credentials
- Enable Symlinks on Windows 10 (Refer to trouble shooting section)


# Usage

Clone this repo and from the root folder with the Vagrantfile file run the following bring up the VM:
```
vagrant up
```

The parent directory of this project is mapped to /vagrant in the VM. (If you have a code base that has windows line 
endings see the trouble shooting section about end of line characters below.)
```
cd /vagrant
```

## Passing in parameters to vagrant

Parameters can be passed into vagrant using environment variables. The example below passes in an ansible tag to 
restrict ansible to running only tasks tagged with 'docker' (ANSIBLE_TAGS=docker). It also tells vagrant to only run 
the ansible provisioner in Vagrantfile (--provision-with ansible). Check out the Vagrant site for more info 
https://www.vagrantup.com/intro/index.html.

```
ANSIBLE_TAGS=docker vagrant provision --provision-with ansible
```

## Cloud Nuke

Cloud nuke (https://github.com/gruntwork-io/cloud-nuke) requires setting the aws profile environment variable in the VM 
to select the profile with the credentials to use.
```
export AWS_PROFILE=<PROFILE NAME>
```

## Gradle tasks

The build-vagrant.gradle file contains gradle convenience tasks for vagrant that can be run. For example:
```
./gradlew vagrantreloadmax
```

# Testing

The Vagrantfile and ansible playbook can be tested by cloning this repository and running vagrant up. ansible/tests.yml 
contains a playbook with tests that validate the tools installed by the ansible/playbook.yml.

```
vagrant up
```

After making changes to the vagrant file or ansible code run an end to end test on the running virtual machine with.

```
vagrant provision
```

## Testing ansible playbooks

The ansible playbook ansible/playbook.yml and test playbook ansible/tests.yml can be run with: 

```
vagrant provision --provision-with ansible,ansible_tests
```

Individual or groups of ansible tests can be run using ansible tags e.g. the following runs only the the ssh config test 
(ANSIBLE_TAGS=testsshconfig) by passing the ansible tag value through vagrant. 

```
ANSIBLE_TAGS=testsshconfig vagrant provision --provision-with ansible_tests
```

Note: The tests are run as a seperate ansible_local vagrant provisioner to allow tasks that require a new shell session 
to take effect.

# Trouble shooting

## Windows Virtualbox error
The following registry entries need to be run from an admin console to get around the error message:

```
Failed to open a session for the virtual machine ubuntu.

Raw-mode is unavailable courtesy of Hyper-V. (VERR_SUPDRV_NO_RAW_MODE_HYPER_V_ROOT).

Result Code: E_FAIL (0x80004005)
Component: ConsoleWrap
Interface: IConsole {872da645-4a9b-1727-bee2-5585105b9eed}
```

```
REG DELETE "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard" /v "EnableVirtualizationBasedSecurity" /f
&& REG DELETE "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard" /v "RequirePlatformSecurityFeatures" /f
&& REG DELETE "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard" /v "Locked" /f
&& REG DELETE "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\HypervisorEnforcedCodeIntegrity" /f
&& REG DELETE "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" /v "LsaCfgFlags" /f
```

or run the gradle task deleteRegistriesForVirtualBox with the gradlew.bat wrapper from an admin commandline on windows.

```
cd localdev
gradlew.bat deleteRegistriesForVirtualBox
```

## End of line characters

Because you're working between linux and windows ensure that you use the LF end of line character with git by having the 
following in your ~/.gitconfig file.

```
[core]
        autocrlf = input
```

To convert windows line endings to linux see: https://askubuntu.com/questions/803162/how-to-change-windows-line-ending-to-unix-version

## Enable Symlinks on Windows

Symlinks creation is disabled by default in windows. You'll need to enable this to do terraform init in the guest 
virtual machine. Instructions available at https://community.perforce.com/s/article/3472


## Switching from virtualbox to vmware

Make sure you vagrant destroy the virtualbox VM first to remove the vboxnet interface otherwise vmware will not be able 
to recreate the new one.


License
-------

BSD


Author Information
------------------

Vang Nguyen

