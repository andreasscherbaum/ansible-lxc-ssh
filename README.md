# ansible-lxc-ssh
Ansible connection plugin using ssh + lxc-attach

![GitHub Workflow Status](https://github.com/andreasscherbaum/ansible-lxc-ssh/actions/workflows/test.yml/badge.svg)
![GitHub Workflow Status](https://github.com/andreasscherbaum/ansible-lxc-ssh/actions/workflows/black.yml/badge.svg)

[![GitHub Open Issues](https://img.shields.io/github/issues/andreasscherbaum/ansible-lxc-ssh.svg)](https://github.com/andreasscherbaum/ansible-lxc-ssh/issues)
[![GitHub Stars](https://img.shields.io/github/stars/andreasscherbaum/ansible-lxc-ssh.svg)](https://github.com/andreasscherbaum/ansible-lxc-ssh)
[![GitHub Forks](https://img.shields.io/github/forks/andreasscherbaum/ansible-lxc-ssh.svg)](https://github.com/andreasscherbaum/ansible-lxc-ssh)

## Description

This plugin allows to use Ansible on a remote server hosting LXC containers,
without having to install SSH servers in each LXC container.

The plugin connects to the host using SSH, then uses `lxc` or `lxc-attach` to enter the
container.

For LXC version 1 this means the SSH connection must login as `root`, otherwise
`lxc-attach` will fail.

For LXC version 2 this means that the user must either login as `root` or must be
in the `lxc` group in order to execute the `lxc` command.

If you are looking for Proxmox support, there's a fork: [ansible-pct-ssh](https://github.com/jbrubake/ansible-pct-ssh):

## Configuration

Add to `ansible.cfg`:
```
[defaults]
connection_plugins = /path/to/connection_plugins/lxc_ssh
```

Then, modify your `hosts` file to use the `lxc_ssh` transport:
```
container ansible_host=server ansible_connection=lxc_ssh lxc_host=container
```

`lxc_container=container` also works for setting the LXC container name.

## Fork

This is a fork from the original plugin:

[ansible-lxc-ssh by Pierre Chifflier](https://github.com/chifflier/ansible-lxc-ssh)

This fork incorporates a few PRs from the original version, which (April 2017) were never
applied. It also works with LXC version 1 (using `lxc-*`) and LXC version 2 (just using
a single `lxc` binary). The version is autodetected on runtime.

## How to create a container

The following is an extract from a Playbook which creates a container. First the hosts.cfg:

```
[containers]
web ansible_host=physical.host lxc_host=web
```

The Playbook:

```
# deploy the container
- hosts: containers
  become: yes
  # the container is not up, nothing to gather here
  gather_facts: False
  # files on the host system are changed,
  # creating multiple containers in parallel might cause a race condition
  serial: 1

  tasks:
  - name: Create LXD Container
    become: True
    lxd_container:
      name: "{{ inventory_name }}"
      state: started
      source:
        type: image
        mode: pull
        server: https://cloud-images.ubuntu.com/releases
        protocol: simplestreams
        alias: 16.10/amd64
      profiles: ['default']
      wait_for_ipv4_addresses: true
      timeout: 600
    register: container_setup
    delegate_to: "{{ ansible_host }}"
    #delegate_facts: True
```

The actual container creation is redirected to the `ansible_host`, also fact gathering is turned off because the container is not yet live. It might be a good idea to create the containers one by one, hence the serialization. In my case I also setup ssh access and hostname resolution during the container setup - this does not work well when run in parallel for multiple containers.
