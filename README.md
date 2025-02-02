# Ansible Role: NetworkConfig

This role configures a Network Interface on Debian based OS. RHEL based OS are also supported. It can be used particularly to assign a fixed IP address to an existing Network Interface.

## Requirements

Any Debian or RHEL based Virtual Machine or Physical server where the Ansible user has SUDO permissions, and python3 is installed.

## Role Variables
For a complete list please see defaults/main.yml.
```
# Network Fixed IP Config
nw_conn_name: "System eth2"
nw_ifname: "eth2"
nw_ip4: "192.168.0.21"
nw_ip4_cidr: "{{ nw_ip4 }}/24"
nw_netmask: "255.255.255.0"
nw_gw4: "192.168.0.1"
nw_dns4_list:
  - "192.168.0.1"
  - "8.8.8.8"
```

## Dependencies
None

## Example Playbook
```
    - hosts: servers
      roles:
         - role: hammadrauf.networkconfig
```
You may have to split your playbook into 2 playbooks and use them as below:
```
- import_playbook: A-Change-Network-Config.yml
- import_playbook: B-Connect-using-new-ip.yml
```

## Detailed Example Usage
  
Vagrant File - ./Vagrantfile
```
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

    config.vm.define "debian" do |debian|
      debian.vm.box = "generic/debian12"
      debian.vm.network "private_network", ip: "192.168.56.6"
      debian.vm.network "public_network", bridge: "Realtek Gaming 2.5GbE Family Controller"
    end
  
    config.vm.provision "file", source: "/home/#{ENV['USER']}/.ssh/id_rsa.pub", destination: "~/.ssh/me.pub"
    config.vm.provision :ansible do |ansible|
      ansible.playbook = "main.yml"
      ansible.raw_arguments = [
       "--vault-password-file=./vars/.vault_pass"
      ]
    end
  end
  
```
  
Main Playbook - ./main.yml:
```
---
###########################
# Ansible Playbook: Roles - main.yml
#   Creates WebNode by orchestrating diffrent roles and tasks.
# Source Repository: https://github.com/????
###########################

- name: Get 1st playbook and play it.
  ansible.builtin.import_playbook: A_pre_ip_change.yml

- name: Get 2nd playbook and play it.
  ansible.builtin.import_playbook: B_post_ip_change.yml
```
  
Playbook 'A' - ./A_pre_ip_change.yml:
```
---
###########################
# Ansible Playbook: Roles - A_pre_ip_change.yml
#   Creates WebNode by orchestrating diffrent roles and tasks.
# Source Repository: https://github.com/????
###########################
- name: Playbook Test Roles - A_pre_ip_change.yml
  hosts: debian
  vars_files:
    - ./vars/vars.yml
    - ./vars/secrets.yml

  tasks:
    - name: Block to include roles as user root
      block:
        - name: Perform Sudouser role tasks
          include_role:
            name: sudousers

        - name: Ensure Ansible tmp directory exists with correct permissions for "sudo su unprevilged"
          ansible.builtin.file:
            path: "/home/{{ su_vault_vmuser1 }}/.ansible/tmp"
            state: directory
            mode: '1777'
            owner: "{{ su_vault_vmuser1 }}"
            group: "{{ su_vault_vmuser1 }}"

        - name: Include Configure-Network Static IP tasks
          include_role:
            name: networkconfig
      become: true

    - name: Flush handlers
      ansible.builtin.meta: flush_handlers
```
  
Playbook 'B' - ./B_post_ip_change.yml:
```
---
###########################
# Ansible Playbook: Roles - B_post_ip_change.yml
#   Creates WebNode by orchestrating diffrent roles and tasks.
# Source Repository: https://github.com/????
###########################
- name: Playbook Test Roles - B_post_ip_change.yml
  hosts: debiannewip
  remote_user: "adminuser"
  vars_files:
    - ./vars/vars.yml
    - ./vars/secrets.yml

  tasks:
    - name: Block to include roles as user root
      block:

        - name: Perform Apache/Httpd role tasks
          include_role:
            name: apache2

      become: true

```
  
Inventory - ./inventory.ini:
```
[debian]
192.168.56.6

[debiannewip]
192.168.0.21

[all:children]
debian
debiannewip
```
  
Ansible CFG - ./ansible.cfg:
```
[defaults]
inventory=./inventory.ini
vault_password_file=./vars/.vault_pass
ansible_ssh_private_key_file="/home/{{ lookup('env', 'USER') }}/.ssh/id_rsa"
nocows=true
# following 2 lines to allow proper handling of temporary files in "sudo su unpreviliged"
allow_world_readable_tmpfiles = true
pipelining = true
enable_task_debugger = false
```
  
Vars File - ./vars/vars.yml:
```
---
####
# Ansible Playbook Variables:
#     - Can include Role default variables that need to be overridden by this playbook.
#     - Secret Variables are saved 'secrets.yml' in seperate file and should not be
#       version controlled. Also they should be encrypted for safety using
#       ansible-vault.
#     - See ../docs sub-folder for help on how to use ansible-vault.
####

# Vars for Role: hammadrauf.sudousers
su_users:
  - username: "{{ su_vault_vmuser1 }}"
    password: "{{ su_vault_vmpwd1 }}"
    is_super_user: true
    sudo_rules: []
  - username: "{{ su_vault_vmuser2 }}"
    password: "{{ su_vault_vmpwd2 }}"
    is_super_user: false
    sudo_rules:
      - "ALL=(ALL)   NOPASSWD: /usr/bin/su - {{ su_vault_vmuser1 }}"
      - "ALL=(root)   NOPASSWD: /bin/su - {{ su_vault_vmuser1 }}"

# Network Fixed IP Config
nw_conn_name: "System eth2"
nw_ifname: "eth2"
nw_ip4: "192.168.0.21"
nw_ip4_cidr: "{{ nw_ip4 }}/24"
nw_netmask: "255.255.255.0"
nw_gw4: "192.168.0.1"
nw_dns4_list:
  - "192.168.0.1"
  - "8.8.8.8"

# Vars for Role: hammadrauf.apache2    
ap2_vapache_default_content: "/var/www/html"
ap2_template_index: "script-uploaded-files/index.html.j2"
ap2_file_css: "script-uploaded-files/site_styles.css"
ap2_file_ico: "script-uploaded-files/beach-ball.ico"
ap2_vapache_remove_default: false
ap2_list_custom_content_folders:
  - /var/www/somesite
  - /var/www/hahahoho
```
  
Secrets File - ./var/secrets.yml
```
su_vault_vmuser1: "adminuser"
su_vault_vmpwd1: xxxx
su_vault_vmuser2: xxxx
su_vault_vmpwd2: xxxx
```

Ansible Vault Password File - ./vars/.vault_pass
```
SomeStrongPassword
```

## Testing with Molecule
Launch Podman instances outside of Molecule/Ansible by using the following commands:
```
podman run -d --name debian12 --hostname debian12 -it docker.io/hammadrauf/dockerdeb12:latest sleep infinity & wait
podman run -d --name fedora40 --hostname fedora40 -it docker.io/hammadrauf/fedora40:latest
podman run -d --name ubuntu --hostname ubuntu -it docker.io/hammadrauf/ubuntunoble:latest sleep infinity & wait
```

## License
MIT

## Author Information
This role was created on Feb 1, 2025 by [Hammad Rauf](https://www.linkedin.com/in/hammadrauf/).
